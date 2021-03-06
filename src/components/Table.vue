<template>
  <div ref="willtable" class="km-willtable">
    <div
      v-if="store.states.columns.length > 0"
      ref="wrapper"
      class="km-table-wrapper"
      :class="{ scrollX: tableWidth > wrapperWidth }"
      :style="{ maxWidth: `${tableWidth}px` }"
      v-clickoutside="clickoutside">
      <div class="km-table-header" ref="theader">
        <table-header
          ref="theaderContent"
          :showIcon="showIcon"
          :columnsWidth="columnsWidth"
          :fixedCount="fixedCount"
          :all-show="true"
          :store="store" />
      </div>
      <div ref="tbody" class="km-table-body" :style="{maxHeight: `${maxHeight}px`}">
        <table-body
          ref="tbodyContent"
          :columnsWidth="columnsWidth"
          :all-show="true"
          :cellStyle="cellStyle"
          :cellClassName="cellClassName"
          :rowHeight="rowHeight"
          :store="store">
          <!-- 编辑器 -->
          <editor
            ref="editor"
            :columnsWidth="columnsWidth"
            :fixedCount="fixedCount"
            :rowHeight="rowHeight"
            :store="store" />
        </table-body>
        <div
          v-if="store.states.showData.length == 0"
          class="empty-block"
          :style="{width: `${tableWidth}px`}">
            暂无数据
        </div>
      </div>
      <!-- 左侧固定- -->
      <div ref="fixedWrapper" class="km-table-fixed" :style="{width: `${fixedWidth}px`}">
        <div class="km-table-fixed-header" ref="fixedTheader">
          <table-header
            ref="fixedTheaderContent"
            :showIcon="showIcon"
            :columnsWidth="columnsWidth"
            :fixedCount="fixedCount"
            :store="store" />
        </div>
        <div ref="fixedTbody" class="km-table-fixed-body">
          <table-body
            ref="fixedTbodyContent"
            :columnsWidth="columnsWidth"
            :cellStyle="cellStyle"
            :cellClassName="cellClassName"
            :rowHeight="rowHeight"
            :store="store" />
        </div>
      </div>
    </div>
    <div ref="wrapper" class="empty-columns" v-else>
      <div ref="theader">
        <div ref="theaderContent"></div>
      </div>
      <div ref="tbody">
        <div ref="tbodyContent"></div>
      </div>
      暂无表头
    </div>
    <dropdown
      :columnsWidth="columnsWidth"
      :fixedCount="fixedCount"
      :store="store"
    ></dropdown>
    <div class="km-adjustLine" :style="{ left: `${store.states.adjustLineLeft}px` }" v-show="store.states.adjustLineShow"></div>
  </div>
</template>

<script>
import '../style/reset.scss';
import TableStore from '../store';
import clickoutside from '../directives/clickoutside';
import methods from '../mixins/methods';
import events from '../mixins/events';
import TableHeader from './TableHeader.vue';
import TableBody from './TableBody.vue';
import Editor from './TableEditor.vue';
import Dropdown from './TableDropdown.vue';

export default {
  name: 'vue-willtable',
  directives: { clickoutside },
  components: {
    TableHeader,
    TableBody,
    Editor,
    Dropdown,
  },
  props: {
    columns: {
      type: Array,
      default: () => [],
    },
    value: {
      type: Array,
      default: () => [],
    },
    maxHeight: {
      type: [String, Number],
    },
    rowHeight: {
      type: [String, Number],
      default: 28,
    },
    disabled: {
      type: Boolean,
      default: false,
    },
    showIcon: {
      type: Boolean,
      default: false,
    },
    cellStyle: {
      type: [Object, Function],
      default: () => () => {},
    },
    cellClassName: {
      type: [Object, Function],
      default: () => () => {},
    },
  },
  data() {
    const store = new TableStore();
    return {
      store,
      wrapperWidth: null,
      tableWidth: null,
      theaderHeight: null,
      columnsWidth: [],

      data: [],
      scrollLeftArr: [],
      scrollTopArr: [],
      fixedCount: 0,
      fixedWidth: 0,

      // 鼠标
      mouseX: 0,
      mouseY: 0,
      timer: null,

      excelPos: {},
    };
  },
  watch: {
    value(val) {
      this.data = val;
    },
    data: {
      handler(val) {
        const { states } = this.store;
        if (!states.isOperation) {
          if (states.historyData.length !== states.curHisory) {
            states.historyData = states.historyData.slice(0, states.curHisory);
          }
          states.historyData.push(JSON.stringify(val));
          states.curHisory += 1;
        }
        this.$emit('input', val);
        if (!states.initialData) {
          this.initData();
        }
        this.store.handleFilters();
        this.store.handleChangeData();
        this.store.handleErrors();
        this.handleResize();
      },
      deep: true,
    },
    columns: {
      handler() {
        this.initColumns();
        this.store.handleFilters();
        this.handleResize();
      },
      deep: true,
    },
  },
  mounted() {
    this.init();
  },
  mixins: [methods, events],
  methods: {
    init() {
      if (this.value.length > 0) {
        this.data = this.value;
      }
      this.$refs.tbody.addEventListener('scroll', () => {
        this.$refs.theader.scrollLeft = this.$refs.tbody.scrollLeft;
        this.$refs.fixedTbody.scrollTop = this.$refs.tbody.scrollTop;
        this.store.tableBodyScroll(this.$refs.tbody);
        this.store.states.dropdown.index = null;
      });
      this.store.handleIsMac();
      this.initColumns();
      this.handleResize();
    },
    initColumns() {
      const { states } = this.store;
      const fixedArr = this.columns.filter((item) => item.fixed);
      const unFixedArr = this.columns.filter((item) => !item.fixed);
      states.columns = fixedArr.concat(unFixedArr).map((item) => {
        if (item.width) {
          item.width = parseInt(item.width, 10);
        }
        return { ...item };
      });
      this.columnsWidth = states.columns.map((item) => item.width);
      states.columnsStatusList = states.columns.map((item) => ({
        key: item.key,
        type: item.type,
        list: {},
        sort: '',
      }));
      states.filters = {};
    },
    initData() {
      const { states } = this.store;
      states.data = this.data;
      states.showData = this.data;
      states.dataStatusList = this.data.map(() => ({
        checked: false,
        errors: [],
      }));
      states.initialData = JSON.parse(JSON.stringify(states.data));
      states.historyData = [JSON.stringify(states.data)];
      states.curHisory = 1;
      if (this.$refs.theaderContent) {
        this.$refs.theaderContent.checkedAll = false;
      }
      if (this.$refs.fixedTheaderContent) {
        this.$refs.fixedTheaderContent.checkedAll = false;
      }
      this.initColumns();
    },
    handleResize() {
      const { states } = this.store;
      this.excelPos = this.$refs.willtable.getBoundingClientRect();
      // 获取编辑框可移动范围, X是横轴, Y是竖轴
      states.editor.editorRange = {
        minX: states.columns.filter((item) => item.type === 'selection').length,
        maxX: states.columns.length - 1,
        minY: 0,
        maxY: states.showData.length - 1,
      };

      this.wrapperWidth = this.$refs.wrapper.offsetWidth;

      this.$nextTick(() => {
        // 计算剩余列宽度
        const surplusColumns = states.columns.filter((item) => !item.width);
        let surplusColumnAvg;

        if (!this.$refs.tbodyContent.$el) return;
        if (surplusColumns.length > 0) {
          const surplusWidth = this.wrapperWidth - states.columns.filter((item) => item.width).reduce((total, item) => total + item.width, 0);
          if (surplusWidth > 0) {
            if (this.$refs.tbodyContent.$el.offsetHeight > this.maxHeight) {
              surplusColumnAvg = (surplusWidth - 1 - states.scrollBarWidth) / surplusColumns.length;
            } else {
              surplusColumnAvg = (surplusWidth - 1) / surplusColumns.length;
            }
          }
        }

        // 设置单元格宽度
        states.columns.forEach((column, index) => {
          if (column.width) {
            this.$set(this.columnsWidth, index, column.width);
          } else {
            // 设置单元格最小宽度
            if (surplusColumnAvg > 80) {
              this.$set(this.columnsWidth, index, surplusColumnAvg);
            } else {
              this.$set(this.columnsWidth, index, 80);
            }
          }
        });

        this.scrollLeftArr = this.columnsWidth.map((item, index) => {
          const sum = this.columnsWidth.reduce((total, curVal, curIndex) => {
            if (curIndex <= index) {
              return total + curVal;
            }
            return total;
          }, 0);
          return sum;
        });

        this.scrollTopArr = states.showData.map((item, index) => {
          const sum = states.showData.reduce((total, curVal, curIndex) => {
            if (curIndex <= index) {
              return total + parseInt(this.rowHeight, 10);
            }
            return total;
          }, 0);
          return sum;
        });
        this.scrollTopArr.unshift(0);

        this.fixedCount = states.columns.filter((item) => item.fixed).length;
        this.fixedWidth = this.scrollLeftArr[this.fixedCount - 1] || 0;

        // 如果每列均设置了宽度
        const allWidth = states.columns.every((cell) => cell.width);
        if (allWidth) {
          this.tableWidth = states.columns.map((cell) => cell.width).reduce((a, b) => a + b, 0);
        } else {
          this.tableWidth = this.columnsWidth.reduce((total, cur) => total + cur, 0);
          if (this.tableWidth < this.wrapperWidth) {
            this.tableWidth = this.wrapperWidth;
          }
        }

        this.$refs.theaderContent.$el.style.width = `${this.tableWidth}px`;
        this.$refs.tbodyContent.$el.style.width = `${this.tableWidth}px`;

        // 设置左侧theader高度与tbody距离顶部距离
        this.theaderHeight = this.$refs.theaderContent.$el.offsetHeight;

        // 设置左侧定位高度
        this.$nextTick(() => {
          if (this.tableWidth > this.wrapperWidth) {
            this.$refs.fixedWrapper.style.height = `${this.$refs.wrapper.offsetHeight - states.scrollBarWidth}px`;
            this.$refs.fixedTbody.style.height = `${this.$refs.wrapper.offsetHeight - states.scrollBarWidth - this.theaderHeight}px`;
          } else {
            this.$refs.fixedWrapper.style.height = `${this.$refs.wrapper.offsetHeight}px`;
            this.$refs.fixedTbody.style.height = `${this.$refs.wrapper.offsetHeight - this.theaderHeight}px`;
          }
        });
        // 当出现竖向滚动条时处理滚动条
        if (this.$refs.tbodyContent.$el.offsetHeight > this.maxHeight) {
          this.$refs.tbody.style.overflowY = 'auto';
          this.$nextTick(() => {
            const colgroup = this.$refs.theaderContent.$el.querySelector('colgroup');
            const tr = this.$refs.theaderContent.$el.querySelector('tr');

            // 如果已存在Col，删除组中重新在末尾添加
            for (let i = 0; i < colgroup.children.length; i += 1) {
              if (colgroup.children[i].width === states.scrollBarWidth) {
                colgroup.removeChild(colgroup.children[i]);
                tr.removeChild(tr.children[i]);
              }
            }
            const col = document.createElement('col');
            col.width = states.scrollBarWidth;
            colgroup.appendChild(col);

            const th = document.createElement('th');
            th.style.width = `${states.scrollBarWidth}px`;
            th.style.borderTop = '1px solid #d6dfe4';
            tr.appendChild(th);

            this.$refs.tbodyContent.$el.style.width = `${this.tableWidth - states.scrollBarWidth - 1}px`;
          });
        }
      });
    },
    clickoutside() {
      const { states } = this.store;
      if (states.selector.isSelected || states.autofill.isAutofill) return;
      states.editor.editing = false;
      states.editor.editorShow = false;
      states.selector.selectedXArr = [];
      states.selector.selectedYArr = [];
      window.removeEventListener('keydown', this.keySubmit);
      window.removeEventListener('mousemove', this.multiSelectAdjustPostion);
    },
    // 选择单元格
    selectCell(e, x, y, type) {
      if (this.disabled) return;
      if (e.button !== 0) return;
      const { states } = this.store;
      window.addEventListener('keydown', this.keySubmit);
      window.addEventListener('mousemove', this.multiSelectAdjustPostion);
      if (type === 'selection') return;
      this.timer = setInterval(this.multiSelectAdjustPostion, 16);
      window.addEventListener('mouseup', this.selectUp);
      states.editor.editing = false;
      states.editor.editType = 'text';
      states.editor.editorShow = true;
      states.autofill.autofillXIndex = x;
      states.autofill.autofillYIndex = y;
      states.selector.selectedXIndex = x;
      states.selector.selectedYIndex = y;
      states.selector.selectedXArr = [x, x];
      states.selector.selectedYArr = [y, y];
      states.selector.isSelected = true;
      this.$nextTick(() => {
        states.editor.editorXIndex = x;
        states.editor.editorYIndex = y;
        states.editor.curEditorCoverValue = states.showData[states.editor.editorYIndex][states.columns[states.editor.editorXIndex].key];
        this.$nextTick(() => {
          this.adjustPosition();
          this.$refs.editor.$refs.clipboard.focus();
        });
      });
    },
    keySubmit(e) {
      const { states } = this.store;
      const { keyCode, ctrlKey, metaKey } = e;
      if ((ctrlKey && keyCode === 90) || (metaKey && keyCode === 90)) {
        return this.store.operation('undo');
      }
      if ((ctrlKey && keyCode === 89) || (metaKey && keyCode === 89)) {
        return this.store.operation('recovery');
      }
      if (states.editor.editing && (keyCode === 13 || keyCode === 27)) {
        states.editor.editing = false;
        states.editor.editType = 'text';
        return;
      }
      if (states.editor.editing || !states.editor.editorShow) {
        return;
      }
      if ((ctrlKey && keyCode === 67) || (metaKey && keyCode === 67)) {
        return this.getContentToclipboard();
      }
      if ((ctrlKey && keyCode === 65) || (metaKey && keyCode === 65)) {
        return this.store.selectAllCells();
      }
      if (ctrlKey || metaKey) {
        return;
      }
      e.preventDefault();
      const isImportability = (k) => {
        if ((k >= 65 && k <= 90) || (k >= 48 && k <= 57) || (k >= 96 && k <= 107) || (k >= 109 && k <= 111) || k === 32 || (k >= 186 && k <= 222)) {
          return true;
        }
        return false;
      };
      switch (true) {
        case keyCode === 37:
          // 左
          if (states.editor.editorXIndex === states.editor.editorRange.minX) {
            states.editor.editorXIndex = states.editor.editorRange.minX;
          } else {
            states.editor.editorXIndex -= 1;
          }
          this.adjustPosition();
          break;
        case keyCode === 38:
          // 上
          if (states.editor.editorYIndex === states.editor.editorRange.minY) {
            states.editor.editorYIndex = states.editor.editorRange.minY;
          } else {
            states.editor.editorYIndex -= 1;
          }
          this.adjustPosition();
          break;
        case keyCode === 39:
          // 右
          if (states.editor.editorXIndex === states.editor.editorRange.maxX) {
            states.editor.editorXIndex = states.editor.editorRange.maxX;
          } else {
            states.editor.editorXIndex += 1;
          }
          this.adjustPosition();
          break;
        case keyCode === 40:
          // 下
          if (states.editor.editorYIndex === states.editor.editorRange.maxY) {
            states.editor.editorYIndex = states.editor.editorRange.maxY;
          } else {
            states.editor.editorYIndex += 1;
          }
          this.adjustPosition();
          break;
        case keyCode === 8:
          // 删除
          this.store.clearSelected();
          break;
        case keyCode === 13:
          this.setEditing();
          break;
        case isImportability(keyCode) || e.key === 'Process' || e.key === 'Unidentified':
          this.setEditing(e.key);
          break;
      }
    },
    getContentToclipboard() {
      const { states } = this.store;
      let content = '';
      for (let i = 0; i <= states.selector.selectedYArr[1] - states.selector.selectedYArr[0]; i += 1) {
        for (let j = 0; j <= states.selector.selectedXArr[1] - states.selector.selectedXArr[0]; j += 1) {
          if (states.selector.selectedXArr[1] - states.selector.selectedXArr[0] === 0) {
            content += `${states.showData[i + states.selector.selectedYArr[0]][states.columns[j + states.selector.selectedXArr[0]].key] || ''}\n`;
          } else if (j === 0) {
            content += (states.showData[i + states.selector.selectedYArr[0]][states.columns[j + states.selector.selectedXArr[0]].key] || '');
          } else if (j === states.selector.selectedXArr[1] - states.selector.selectedXArr[0]) {
            content += `\t${states.showData[i + states.selector.selectedYArr[0]][states.columns[j + states.selector.selectedXArr[0]].key] || ''}\n`;
          } else {
            content += `\t${states.showData[i + states.selector.selectedYArr[0]][states.columns[j + states.selector.selectedXArr[0]].key] || ''}`;
          }
        }
      }
      const textArea = document.createElement('textarea');
      textArea.value = content;
      document.body.appendChild(textArea);
      textArea.select();
      try {
        const successful = document.execCommand('copy');
        console.log(successful);
      } catch (err) {
        console.log(err);
      }
      document.body.removeChild(textArea);
    },
    clipboardToContent(e) {
      setTimeout(() => {
        const arr = [];
        e.target.value.split('\n').forEach((item, index, curArr) => {
          if (this.store.states.isMac) {
            arr.push(item.split('\t'));
          } else {
            if (index < curArr.length - 1) {
              arr.push(item.split('\t'));
            }
          }
        });
        for (let i = 0; i <= arr.length - 1; i += 1) {
          for (let j = 0; j <= arr[i].length - 1; j += 1) {
            if (this.store.states.showData[i + this.store.states.selector.selectedYArr[0]] && this.store.states.columns[j + this.store.states.selector.selectedXArr[0]]) {
              if (!this.store.states.columns[j + this.store.states.selector.selectedXArr[0]].disabled) {
                this.store.states.showData[i + this.store.states.selector.selectedYArr[0]][this.store.states.columns[j + this.store.states.selector.selectedXArr[0]].key] = arr[i][j];
              }
            }
          }
        }
        this.$refs.editor.$refs.clipboard.value = '';
      }, 0);
    },
    // 设置启用编辑
    setEditing(key) {
      const { states } = this.store;
      if (states.columns[states.editor.editorXIndex].disabled) {
        return;
      }
      states.editor.editType = states.columns[states.editor.editorXIndex].type ? states.columns[states.editor.editorXIndex].type : 'text';
      states.editor.curEditorWidth = this.columnsWidth[states.editor.editorXIndex];
      if (key && (states.editor.editType === 'text' || states.editor.editType === 'number')) {
        if (key === 'Process' || key === 'Unidentified') {
          if (key.match(/\d/)) {
            [this.$refs.editor.editContent] = key.match(/\d/);
          } else {
            this.$refs.editor.editContent = '';
          }
        } else {
          this.$refs.editor.editContent = key;
        }
      } else {
        this.$refs.editor.editContent = states.showData[states.editor.editorYIndex][states.columns[states.editor.editorXIndex].key];
      }
      states.editor.editing = true;
      if (states.editor.editType === 'select') {
        states.editor.options = states.columns[states.editor.editorXIndex].options;
      }
      this.$nextTick(() => {
        this.$refs.editor.$refs[states.editor.editType].focus();
      });
    },
    selectUp() {
      clearInterval(this.timer);
      setTimeout(() => {
        this.store.states.selector.isSelected = false;
      }, 0);
    },
    adjustPosition() {
      const { states } = this.store;
      if (states.editor.editorXIndex < this.fixedCount) {
        states.editor.editorIsFixed = true;
      } else {
        states.editor.editorIsFixed = false;
      }
      states.autofill.autofillXIndex = states.editor.editorXIndex;
      states.autofill.autofillYIndex = states.editor.editorYIndex;
      states.selector.selectedXArr = [states.editor.editorXIndex, states.editor.editorXIndex];
      states.selector.selectedYArr = [states.editor.editorYIndex, states.editor.editorYIndex];
      // 左右调整
      let curLeftShould; let
        curRightShould;
      if (this.fixedCount > 0) {
        const unFixedLeftArr = this.scrollLeftArr.slice(this.fixedCount - 1, -1).map((item) => item - this.fixedWidth);
        const unFixedRightArr = this.scrollLeftArr.slice(this.fixedCount).map((item) => item - this.fixedWidth);
        curLeftShould = unFixedLeftArr[states.editor.editorXIndex - this.fixedCount];
        curRightShould = unFixedRightArr[states.editor.editorXIndex - this.fixedCount] + this.fixedWidth - this.wrapperWidth + states.scrollBarWidth + 2;
      } else {
        const scrollLeftArr = [0, ...this.scrollLeftArr];
        curLeftShould = scrollLeftArr[states.editor.editorXIndex];
        curRightShould = scrollLeftArr[states.editor.editorXIndex + 1] - this.wrapperWidth + states.scrollBarWidth + 2;
      }
      if (this.tableWidth > this.wrapperWidth) {
        if (states.tableBodyLeft > curLeftShould) {
          this.$refs.theader.scrollLeft = curLeftShould;
          this.$refs.tbody.scrollLeft = curLeftShould;
        }
        if (states.tableBodyLeft < curRightShould) {
          this.$refs.theader.scrollLeft = curRightShould + 2;
          this.$refs.tbody.scrollLeft = curRightShould + 2;
        }
      }
      // 上下调整
      if (this.maxHeight) {
        const curTopShould = this.scrollTopArr[states.editor.editorYIndex];
        if (states.tableBodyTop > curTopShould) {
          this.$refs.tbody.scrollTop = curTopShould;
          this.$refs.fixedTbody.scrollTop = curTopShould;
        }
        const curBottomShould = this.scrollTopArr[states.editor.editorYIndex + 1] - this.maxHeight + states.scrollBarWidth + 2;
        if (states.tableBodyTop < curBottomShould) {
          this.$refs.tbody.scrollTop = curBottomShould;
          this.$refs.fixedTbody.scrollTop = curBottomShould;
        }
      }
    },
    multiSelectAdjustPostion(e) {
      const { states } = this.store;
      if (states.selector.isSelected) {
        if (e) {
          e = e || window.event;
          this.mouseX = e.pageX;
          this.mouseY = e.pageY;
        }
        if (states.selector.selectedYArr[0] === states.selector.selectedYArr[1] && states.selector.selectedXArr[0] === states.selector.selectedXArr[1]) {
          return;
        }
        const sTop = this.$refs.willtable.offsetTop + 30;
        const sLeft = this.excelPos.left + this.fixedWidth;
        const sBottom = this.$refs.willtable.offsetTop + this.$refs.willtable.offsetHeight - 10;
        const sRight = this.excelPos.left + this.$refs.willtable.offsetWidth - 10;
        if (this.mouseY < sTop) {
          this.$refs.tbody.scrollTop -= 20;
          this.$refs.fixedTbody.scrollTop -= 20;
        }
        if (this.mouseY > sBottom) {
          this.$refs.tbody.scrollTop += 20;
          this.$refs.fixedTbody.scrollTop += 20;
        }
        if (this.mouseX < sLeft) {
          this.$refs.theader.scrollLeft -= 20;
          this.$refs.tbody.scrollLeft -= 20;
        }
        if (this.mouseX > sRight) {
          this.$refs.theader.scrollLeft += 20;
          this.$refs.tbody.scrollLeft += 20;
        }
      }
    },
    selectAll() {
      const { states } = this.store;
      const checkedAll = states.dataStatusList.every((item) => item.checked);
      states.dataStatusList.forEach((item, index) => {
        this.$set(states.dataStatusList[index], 'checked', !checkedAll);
      });
      this.selectionChange();
    },
    adjustWidth(index, width) {
      this.store.states.columns[index].width = width;
      this.handleResize();
    },
  },
};
</script>

<style lang="scss" scoped>
.km-willtable {
  position: relative;
}

div,
section,
ul {
  &::-webkit-scrollbar {
    width: 10px;
    height: 10px;
  }
  &::-webkit-scrollbar-track {
    background-color: #e9eaec;
  }
  &::-webkit-scrollbar-thumb {
    background-color: #bbbec4;
    border-radius: 10px;
    transition: all 1s;
  }
  &::-webkit-scrollbar-thumb:hover {
    background-color: #80848f;
  }
}

.km-table-wrapper {
  position: relative;
  overflow: hidden;
  &.scrollX {
    border-right: 1px solid #d6dfe4;
    border-left: 1px solid #d6dfe4;
    /deep/ th,
    /deep/ td {
      &:first-child {
        border-left: 0;
      }
      &:last-child {
        border-right: 0;
      }
    }
    .km-table-body {
      overflow-x: auto;
    }
  }
  /deep/ table {
    border-spacing: 0;
    border-collapse: collapse;
    font-size: 12px;
    table-layout: fixed;
    .km-cell-content {
      width: 100%;
      overflow: hidden;
      white-space: nowrap;
      text-overflow: ellipsis;
    }
  }
}

.km-table-header {
  overflow: hidden;
  /deep/ table {
    border-bottom: 1px solid #d6dfe4;
  }
}

.km-table-body {
  overflow: hidden;
  user-select: none;
  border-bottom: 1px solid #d6dfe4;

  /deep/ tbody {
    tr:first-child {
      td {
        border-top: none;
      }
    }
  }
}

// 左侧固定
.km-table-fixed {
  position: absolute;
  top: 0;
  left: 0;
  box-shadow: 1px 0 8px #d3d4d6;
  overflow-x: hidden;
  background: #fff;
  user-select: none;
  z-index: 2;
  .km-table-fixed-header {
    position: absolute;
    top: 0;
    left: 0;
    z-index: 3;
    /deep/ table {
      border-bottom: 1px solid #d6dfe4;
    }
  }
  .km-table-fixed-body {
    position: absolute;
    top: 30px;
    left: 0;
    z-index: 3;
    overflow: hidden;
    border-bottom: 1px solid #d6dfe4;
    /deep/ tbody {
      tr:first-child {
        td {
          border-top: 2px solid transparent;
        }
      }
    }
  }
}

// 数据为空
.empty-block {
  position: relative;
  z-index: 9;
  font-size: 14px;
  padding: 30px;
  text-align: center;
  border-right: 1px solid #d6dfe4;
  border-left: 1px solid #d6dfe4;
  color: #909399;
}

.empty-columns {
  text-align: center;
  border: 1px solid #DCDFE6;
  padding: 10px 20px;
  color: #909399;
}

.el-checkbox {
  font-size: 12px;
}

.km-adjustLine {
  position: absolute;
  top: 0;
  left: 0;
  width: 1px;
  height: 100%;
  background-color: #d6dfe4;
  z-index: 10;
}
</style>
