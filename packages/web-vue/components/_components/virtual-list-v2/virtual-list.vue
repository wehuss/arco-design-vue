<template>
  <div ref="containerRef" :class="prefixCls" :style="style" @scroll="onScroll">
    <div
      ref="contentRef"
      :style="{
        paddingTop: `${frontPadding}px`,
        paddingBottom: `${behindPadding}px`,
      }"
    >
      <VirtualListItem
        v-for="(item, index) of currentList"
        :key="item[itemKey]"
        :has-item-size="hasItemSize"
        :set-item-size="setItemSize"
      >
        <slot name="item" :item="item" :index="start + index" />
      </VirtualListItem>
    </div>
  </div>
</template>

<script lang="ts">
import {
  computed,
  CSSProperties,
  defineComponent,
  nextTick,
  onMounted,
  onUpdated,
  PropType,
  ref,
  toRefs,
} from 'vue';
import './style/index.less';
import { useSize } from './hooks/use-size';
import VirtualListItem from './virtual-list-item';
import { getPrefixCls } from '../../_utils/global-config';
import { ScrollOptions } from './interface';
import { isNumber } from '../../_utils/is';

export default defineComponent({
  name: 'VirtualList',
  components: { VirtualListItem },
  props: {
    height: {
      type: [Number, String],
      default: 200,
    },
    data: {
      type: Array,
      default: () => [],
    },
    itemKey: {
      type: String,
      default: 'key',
    },
    fixedSize: {
      type: Boolean,
      default: false,
    },
    estimatedSize: {
      type: Number,
    },
    buffer: {
      type: Number,
      default: 10,
    },
  },
  emits: {
    scroll: () => true,
    reachBottom: () => true,
  },
  setup(props, { emit }) {
    const { data, itemKey, fixedSize, estimatedSize, buffer, height } =
      toRefs(props);
    const prefixCls = getPrefixCls('virtual-list');

    const containerRef = ref<HTMLElement>();
    const contentRef = ref<HTMLElement>();

    const style = computed(() => {
      return {
        height: isNumber(height.value) ? `${height.value}px` : height.value,
      };
    });

    const dataKeys = computed(() =>
      data.value.map((item: any) => {
        return item[itemKey.value] as string | number;
      })
    );

    const {
      frontPadding,
      behindPadding,
      start,
      end,
      getStartByScroll,
      setItemSize,
      hasItemSize,
      setStart,
      getScrollOffset,
    } = useSize({
      dataKeys,
      contentRef,
      fixedSize,
      estimatedSize,
      buffer,
    });

    const currentList = computed(() => {
      return data.value.slice(start.value, end.value);
    });

    const onScroll = (ev: Event) => {
      const { scrollTop, scrollHeight, offsetHeight } =
        ev.target as HTMLElement;

      const _start = getStartByScroll(scrollTop);
      if (_start !== start.value) {
        start.value = _start;
      }
      emit('scroll');
      const bottom = Math.floor(scrollHeight - (scrollTop + offsetHeight));
      if (bottom <= 0) {
        emit('reachBottom');
      }
    };

    const scrollTo = (options: ScrollOptions) => {
      if (containerRef.value) {
        if (isNumber(options)) {
          containerRef.value.scrollTop = options;
        } else {
          const { align = 'top' } = options;
          const _index =
            options.index ?? dataKeys.value.indexOf(options.key ?? '');
          setStart(_index - buffer.value);
          containerRef.value.scrollTop = getScrollOffset(_index);
          nextTick(() => {
            if (containerRef.value) {
              const _scrollTop = getScrollOffset(_index);
              if (_scrollTop !== containerRef.value.scrollTop) {
                containerRef.value.scrollTop = _scrollTop;
              }
            }
          });
        }
      }
    };

    return {
      prefixCls,
      containerRef,
      contentRef,
      frontPadding,
      currentList,
      behindPadding,
      onScroll,
      setItemSize,
      hasItemSize,
      start,
      scrollTo,
      style,
    };
  },
});
</script>
