<template>
  <div>
    <LoadingAndError :loading="loading" :error="error" />

    <AddProcessorModal
      :is-visible="addProcessorModalIsVisible"
      :modules="modules"
      @add="addProcessor"
      @close="addProcessorModalIsVisible = false"
    />

    <n-form v-if="form" ref="formRef" :model="form" :rules="rules as any">
      <n-grid :cols="24" :x-gap="24">
        <n-form-item-gi :span="12" label="标识名称" path="name">
          <n-input v-model:value="form.name" placeholder="请输入唯一标识名称" />
        </n-form-item-gi>
        <n-form-item-gi :span="12" label="展示名称" path="displayName">
          <n-input
            v-model:value="form.displayName"
            placeholder="请输入订阅展示名称"
          />
        </n-form-item-gi>
      </n-grid>

      <n-grid :cols="24" :x-gap="24">
        <n-form-item-gi :span="18" label="订阅图标" path="icon">
          <n-input
            v-model:value="form.icon"
            placeholder="请输入订阅图标地址或 base64 字符串，推荐 png/webp 等包含 alpha 通道的格式"
          />
        </n-form-item-gi>
        <n-grid-item :span="6" class="flex items-center gap-x-[8px]">
          图标预览：
          <AutoImage :src="form.icon" height="64px" />
        </n-grid-item>
      </n-grid>

      <!-- 单条订阅表单项目 -->
      <template v-if="props.type === 'sub'">
        <n-form-item label="订阅类型" path="source">
          <n-radio-group v-model:value="(form as Subscription.Sub).source">
            <n-radio-button value="remote" label="远程" />
            <n-radio-button value="local" label="本地" />
          </n-radio-group>
        </n-form-item>

        <template v-if="(form as Subscription.Sub).source === 'remote'">
          <n-form-item label="订阅地址" path="url">
            <n-input
              v-model:value="(form as Subscription.Sub).url"
              placeholder="请输入 NodeList 订阅地址"
            />
          </n-form-item>

          <n-form-item label="User-Agent" path="ua">
            <n-input
              v-model:value="(form as Subscription.Sub).ua"
              placeholder="请输入拉取订阅使用的User-Agent"
            />
          </n-form-item>
        </template>
        <template v-if="(form as Subscription.Sub).source === 'local'">
          <n-form-item label="节点内容" path="content">
            <n-input
              v-model:value="(form as Subscription.Sub).content"
              type="textarea"
              placeholder="请输入节点信息，支持格式请查看"
              :autosize="{
                minRows: 5,
                maxRows: 12,
              }"
            />
          </n-form-item>
        </template>
      </template>

      <!-- 组合订阅表单项目 -->
      <template v-if="props.type === 'collection'">
        <n-form-item label="包含子订阅" path="subscriptions">
          <n-select
            v-model:value="(form as Subscription.Collection).subscriptions"
            multiple
            filterable
            :options="
              subs.map((s) => ({
                value: s.name,
                label: s.displayName || s.name,
              }))
            "
            :render-label="renderSubscriptionsLabel"
            placeholder="请选择包含的单条订阅"
          />
        </n-form-item>
      </template>

      <n-divider />

      <h2 class="flex items-center gap-x-[16px] pc-secondary-title">
        <span>订阅处理</span>
        <n-button size="small" @click="addProcessorModalIsVisible = true">
          <template #icon>
            <i class="i-carbon-add block" />
          </template>
          添加处理器
        </n-button>
      </h2>

      <Processor :value="form.process" @change="onProcessChange" />

      <n-divider />

      <div class="flex justify-end">
        <n-button
          attr-type="submit"
          size="large"
          type="primary"
          @click="submitForm"
        >
          提交
        </n-button>
      </div>
    </n-form>
  </div>
</template>

<script setup lang="ts">
import type { FormInst, SelectOption } from 'naive-ui';
import { useDialog } from 'naive-ui';
import { storeToRefs } from 'pinia';
import type { VNodeChild } from 'vue';
import { h, onMounted, ref } from 'vue';
import { onBeforeRouteLeave } from 'vue-router';

import AutoImage from '../../../components/pc/AutoImage.vue';
import { useApi } from '../../../hooks/useApi.ts';
import { useAppMessage } from '../../../hooks/useAppMessage.tsx';
import { router } from '../../../routes';
import { useModuleStore } from '../../../store/useModuleStore.ts';
import { useSubscriptionStore } from '../../../store/useSubscriptionStore.ts';
import AddProcessorModal from './components/AddProcessorModal.vue';
import Processor from './components/Processor.vue';

const props = defineProps<{
  name?: string;
  type: Components.SubType;
}>();

const emits = defineEmits<{
  (
    e: 'submit',
    data: Subscription.Sub | Subscription.Collection,
  ): Promise<void>;
}>();
const subStore = useSubscriptionStore();
const moduleStore = useModuleStore();
const { subs, collections, isInit: subsIsInit } = storeToRefs(subStore);
const { modules, isInit: modulesIsInit } = storeToRefs(moduleStore);

const loading = ref(false);
const error = ref('');
const form = ref<Subscription.Sub | Subscription.Collection | null>(null);
const formRef = ref<FormInst | null>(null);

// 用来保留旧数据，在路由切换时候校验提示保存数据
let initForm = '';

const randomId = (): number => {
  const id = Math.floor(Math.random() * 1000000);
  return form.value?.process.some(p => p.id === id) ? randomId() : id;
};

const { showAppMessage } = useAppMessage();
// 添加处理器
const addProcessorModalIsVisible = ref(false);
const addProcessor = (name: string) => {
  addProcessorModalIsVisible.value = false;
  if (!form.value) return;
  const module = modules.value.find(m => m.name === name);
  if (!module) {
    return showAppMessage({
      type: 'error',
      message: '未知模块',
    });
  }

  let content = `/api/module/${name}`;
  if (module.params) {
    const paramsObj: Record<string, any> = {};
    for (const key in module.params) {
      paramsObj[key] = module.params[key].defaultValue ?? null;
    }
    content += `#${JSON.stringify(paramsObj)}`;
    content = encodeURI(content);
  }

  form.value.process.push({
    type: 'Script Operator',
    args: {
      content,
      mode: 'link',
    },
    id: randomId(),
  });
};

// 修改处理器数据
const onProcessChange = (list: Subscription.Processors) => {
  if (!form.value) return;
  form.value.process = [...list];
};

// 组合订阅的单条订阅选项渲染函数
const renderSubscriptionsLabel = (option: SelectOption): VNodeChild =>
  h('div', { class: 'flex gap-x-[8px]' }, [
    h(AutoImage, {
      src: subs.value.find(sub => sub.name === option.value)?.icon,
      height: '20px',
    }),
    option.label as string,
  ]);

const rules = {
  name: [
    {
      required: true,
      validator: (_: any, value: string) => {
        const list = props.type === 'sub' ? subs.value : collections.value;
        if (!value) {
          return new Error('请输入唯一标识名称');
        } else if (!/^[a-zA-Z0-9_-]+$/.test(value)) {
          return new Error('唯一标识名称只能包含字母、数字、下划线和中划线');
        } else if (!props.name && list.some(sub => sub.name === value)) {
          return new Error('唯一标识名称已存在，请更换');
        }
        return true;
      },
      trigger: ['input', 'blur'],
    },
  ],
  displayName: [
    {
      required: true,
      message: '请输入订阅展示名称',
      trigger: ['input', 'blur'],
    },
  ],
  icon: [
    {
      validator: (_: any, value: string) => {
        if (!value) {
          return true;
        } else if (
          !/^https?:\/\/.+$/.test(value) &&
          !/^data:image.+$/.test(value)
        ) {
          return new Error('图标地址应以 http(s)?:// 或 data:image 开头');
        }
        return true;
      },
      trigger: ['input', 'blur'],
    },
  ],
  url: [
    {
      required: true,
      validator: (_: any, value: string) => {
        if (!value) {
          return new Error('请输入 NodeList 订阅地址');
        } else if (!/^https?:\/\/.+$/.test(value)) {
          return new Error('订阅地址应以 http(s)?:// 开头');
        }
        return true;
      },
      trigger: ['input', 'blur'],
    },
  ],
  content: [
    {
      required: true,
      message: '请输入节点信息',
      trigger: ['input', 'blur'],
    },
  ],
  subscriptions: [
    {
      required: true,
      type: 'array',
      message: '请选择包含的单条订阅',
      trigger: ['change', 'blur'],
    },
  ],
};

const submitForm = (e: MouseEvent) => {
  e.preventDefault();
  formRef.value?.validate((errors) => {
    if (!errors) {
      // 防止弹窗，将对比值设置为当前值
      initForm = JSON.stringify({ ...form.value });
      emits('submit', form.value!);
    } else {
      console.log(errors);
    }
  });
};

const dialog = useDialog();
const { subApi, moduleApi } = useApi();

onMounted(async () => {
  loading.value = true;

  if (!modulesIsInit.value) {
    const modules = await moduleApi.getModules();
    moduleStore.setModules(modules);
  }

  if (!subsIsInit.value) {
    const [subs, collections] = await Promise.all([
      subApi.getSubs(),
      subApi.getCollections(),
    ]);
    subStore.setSubs(subs);
    subStore.setCollections(collections);
  }

  if (props.name) {
    if (props.type === 'sub') {
      const sub = subs.value.find(sub => sub.name === props.name);
      if (sub) {
        form.value = { ...sub };
      } else {
        error.value = '未找到该订阅';
      }
    } else {
      const collection = collections.value.find(
        collection => collection.name === props.name,
      );
      if (collection) {
        form.value = { ...collection };
      } else {
        error.value = '未找到该集合';
      }
    }

    if (form.value) {
      const hasOldProcess = form.value.process.some(
        p =>
          (p.type === 'Script Operator' && p.args.mode !== 'link') ||
          p.type !== 'Script Operator',
      );
      if (hasOldProcess) {
        // 如果含有旧的 process，弹窗提示
        dialog.warning({
          title: '警告',
          content:
            '此订阅包含旧版本订阅处理器，当前版本已全面迁移至模块处理，继续编辑并保存后旧处理器将被全部丢弃。如果依然需要在旧版本 Sub-Store 使用，建议拷贝一份订阅使用新版处理器哦～',
          positiveText: '抛弃旧处理器，使用新版',
          negativeText: '返回',
          closable: false,
          closeOnEsc: false,
          maskClosable: false,
          autoFocus: false,
          onPositiveClick: () => {
            form.value!.process = form.value!.process.filter(
              p => p.type === 'Script Operator' && p.args.mode === 'link',
            );
          },
          onNegativeClick: () => router.back(),
        });
      }
    }
  } else {
    if (props.type === 'sub') {
      form.value = {
        content: '',
        displayName: '',
        icon: '',
        name: '',
        source: 'remote',
        ua: '',
        url: '',
        process: [],
      };
    } else {
      form.value = {
        displayName: '',
        icon: '',
        name: '',
        subscriptions: [],
        process: [],
      };
    }
  }

  loading.value = false;
  initForm = JSON.stringify({ ...form.value });
});

onBeforeRouteLeave((_, __, next) => {
  if (JSON.stringify(form.value) !== initForm) {
    dialog.warning({
      title: '警告',
      content: '当前订阅已修改尚未保存，离开将丢失修改。是否确认离开？',
      positiveText: '离开',
      negativeText: '取消',
      closable: false,
      closeOnEsc: false,
      maskClosable: false,
      autoFocus: false,
      onPositiveClick: () => next(),
    });
  } else {
    next();
  }
});
</script>
