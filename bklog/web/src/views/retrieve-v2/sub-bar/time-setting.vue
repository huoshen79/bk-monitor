<script setup>
  import { computed,ref,watch,onUnmounted} from 'vue';
  import { RetrieveUrlResolver } from '@/store/url-resolver';

  import TimeRange from '@/components/time-range/time-range';
  import { handleTransformToTimestamp } from '@/components/time-range/utils';
  import useStore from '@/hooks/use-store';
  import { updateTimezone } from '@/language/dayjs';
  import { useRoute, useRouter } from 'vue-router/composables';

  import RetrieveHelper, { RetrieveEvent } from '../../retrieve-helper';

  const store = useStore();
  const route = useRoute();
  const router = useRouter();
  const isDropdownShow = ref(false);
  const dropdown = ref(null);
  const selectedValue = ref('off');
  let refreshTimer = null;
  const datasource = ref([
  { label: '关闭(off)', value: 'off' },
  { label: '10s', value: '10s' }, 
  { label: '30s', value: '30s' }, 
  { label: '1m', value: '1m' }, 
  { label: '5m', value: '5m' }, 
  { label: '15m', value: '15m' }, 
  { label: '30m', value: '30m' }, 
  { label: '1h', value: '1h' }, 
  { label: '2h', value: '2h' }, 
  { label: '1d', value: '1d' }, 
]);

/**
 * 将时间字符串转换为毫秒数
 * @param {string} timeValue
 * @returns {number} 毫秒数
 */
const getTimeInMs = (timeValue) => {
  if (timeValue === 'off') return 0;

  const unitMap = {
    s: 1000, // 秒
    m: 60 * 1000, // 分钟
    h: 60 * 60 * 1000, // 小时
    d: 24 * 60 * 60 * 1000, // 天
  };

  const unit = timeValue.slice(-1);
  const amount = parseInt(timeValue.slice(0, -1), 10);

  // 输入验证
  if (isNaN(amount) || amount <= 0) {
    return 0;
  }

  if (!unitMap[unit]) {
    return 0;
  }

  return amount * unitMap[unit];
};

const handleRefresh = () => {
  setRouteParams();
  // 触发自动刷新事件
  setTimeout(() => { RetrieveHelper.fire(RetrieveEvent.AUTO_REFRESH); })
};

const clearRefreshTimer = () => {
  if (refreshTimer) {
    clearInterval(refreshTimer);
    refreshTimer = null;
  }
};

// 移除了 TypeScript 类型注解
const setRefreshTimer = (timeValue) => {
  clearRefreshTimer();

  if (timeValue === 'off') return;

  const interval = getTimeInMs(timeValue);

  if (interval > 0) {
    refreshTimer = setInterval(handleRefresh, interval);
  }
};

watch(
  selectedValue,
  (newVal) => {
    setRefreshTimer(newVal);
  },
  { immediate: true }
);

onUnmounted(() => {
  clearRefreshTimer();
});

const dropdownShow = () => {
  isDropdownShow.value = true;
};

const dropdownHide = () => {
  isDropdownShow.value = false;
};

const triggerHandler = (item) => {
  selectedValue.value = item.value;
  dropdown.value.hide();
};
  /** 时间选择器绑定的值 */
  const timeRangValue = computed(() => {
    return store.state.indexItem.datePickerValue;
  });

  const formatValue = computed(() => store.getters.retrieveParams.format);

  const setRouteParams = () => {
    const query = { ...route.query };
    const { start_time, end_time, interval, begin, size, format } = store.getters.retrieveParams;
    const timezone = store.state.indexItem.timezone;

    const resolver = new RetrieveUrlResolver({
      start_time,
      end_time,
      interval,
      timezone,
      begin,
      size,
      format,
      datePickerValue: store.state.indexItem.datePickerValue,
    });
    Object.assign(query, resolver.resolveParamsToUrl());
    router.replace({
      query,
    });
  };

  const timezone = computed(() => {
    const { timezone = '' } = store.state.indexItem;
    return timezone;
  });

  const handleTimezoneChange = timezone => {
    store.commit('updateIndexItemParams', { timezone });
    updateTimezone(timezone);
    store.dispatch('requestIndexSetQuery');
    setRouteParams();
  };

  // 日期变化
  const handleTimeRangeChange = async val => {
    store.commit('updateIsSetDefaultTableColumn', false);
    const result = handleTransformToTimestamp(val, formatValue.value);
    store.commit('updateIndexItemParams', { start_time: result[0], end_time: result[1], datePickerValue: val });
    await store.dispatch('requestIndexSetFieldInfo');
    store.dispatch('requestIndexSetQuery');
    setRouteParams();
    RetrieveHelper.fire(RetrieveEvent.SEARCH_TIME_CHANGE, val);
  };

  const handleFormatChange = value => {
    store.commit('updateIndexItemParams', { format: value });
  };
  defineExpose({
    setRouteParams
  })
</script>
<template>
  <span class="query-params-wrap">
    <TimeRange
      :timezone="timezone"
      :value="timeRangValue"
      @change="handleTimeRangeChange"
      @timezone-change="handleTimezoneChange"
      @format-change="handleFormatChange"
    ></TimeRange>
    <span class="custom-border"></span>
     <div class="auto-refresh-sub-bar">
    <bk-dropdown-menu trigger="click" @show="dropdownShow" @hide="dropdownHide" ref="dropdown" ext-cls="dropdown-menu"
      class="auto-refresh-dropdown">
      <div class="dropdown-trigger-text" :class="{ 'active': isDropdownShow }" slot="dropdown-trigger"
        v-bk-tooltips="{ content: $t('自动刷新设置'), placement: 'bottom' }">
        <i class="bklog-icon bklog-auto-refresh"></i>
        <span>{{datasource.find(item => item.value === selectedValue)?.value || 'off'}}</span>
      </div>
      <ul class="bk-dropdown-list" slot="dropdown-content">
        <li v-for="item in datasource" :key="item.value">
          <a href="javascript:;" @click="() => triggerHandler(item)">{{ item.label }}</a>
        </li>
      </ul>
    </bk-dropdown-menu>
  </div>
  </span>
</template>
<style lang="scss" scoped>
@import'./time-setting.scss';
</style>