<template>
  <div class="test-excel">
    <a-upload
      class="upload-wrapper"
      :file-list="fileList"
      :remove="handleRemove"
      :before-upload="beforeUpload"
      accept="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet,"
    >
      <a-button> <a-icon type="upload" /> 选择excel </a-button>
    </a-upload>
    <a-divider>内容显示</a-divider>
    <div class="content-box">
      <a-tabs
        :active-key="activeKey"
        v-if="sheets.length > 0"
        @change="onChange"
      >
        <a-tab-pane v-for="sheet in sheets" :key="sheet.name" :tab="sheet.name">
          <!-- <a-table v-bind="sheet.tableParams" :pagination="false"></a-table> -->
          <vue-virtual-table v-bind="sheet.tableParams"> </vue-virtual-table>
        </a-tab-pane>
      </a-tabs>
      <a-empty v-else />
    </div>
    <a-button class="btn-filter-compare" @click="onFilterCompare"
      >过滤比较</a-button
    >
    <Modal ref="Modal" @save="onSave"></Modal>
  </div>
</template>

<script>
import XLSX from "xlsx";
import NProgress from "nprogress";
import VueVirtualTable from "vue-virtual-table";
import Modal from "./modal.vue";

export default {
  components: { VueVirtualTable, Modal },
  data() {
    console.log(this);
    this.wb = null;
    this.count = 0;
    return {
      fileList: [],
      sheets: [],
      activeKey: "",
    };
  },
  methods: {
    handleRemove(file) {
      this.fileList = [];
      this.sheets = [];
      this.activeKey = "";
    },
    beforeUpload(file) {
      this.fileList = [file];
      this.sheets = [];
      this.activeKey = "";
      this.showExcel(file);
      return false;
    },
    showExcel(file) {
      var binary = "";
      var wb; //读取完成的数据
      var reader = new FileReader();
      reader.onload = (e) => {
        NProgress.done();
        var bytes = new Uint8Array(reader.result);
        var length = bytes.byteLength;
        for (var i = 0; i < length; i++) {
          binary += String.fromCharCode(bytes[i]);
        }
        wb = XLSX.read(binary, {
          type: "binary",
        });
        this.wb = wb;
        this.count = this.count + 1;
        this.sheets = wb.SheetNames.map((item, index) => {
          return {
            name: item,
            tableParams: index == 0 ? this.getTableParams(wb, item) : {},
          };
        });
        this.activeKey = _.get(this.sheets, "0.name");
      };
      setTimeout(() => {
        NProgress.set(0.4);
        reader.readAsArrayBuffer(file);
      }, 500);
    },
    num2Str(n) {
      var stringArray = [];
      stringArray.length = 0;
      var numToStringAction = function (nnum) {
        var num = nnum - 1;
        var a = parseInt(num / 26);
        var b = num % 26;
        stringArray.push(String.fromCharCode(64 + parseInt(b + 1)));
        if (a > 0) {
          numToStringAction(a);
        }
      };
      numToStringAction(n);
      return stringArray.reverse().join("");
    },
    str2Num(a) {
      var str = a.toLowerCase().split("");
      var num = 0;
      var al = str.length;
      var getCharNumber = function (charx) {
        return charx.charCodeAt() - 96;
      };
      var numout = 0;
      var charnum = 0;
      for (var i = 0; i < al; i++) {
        charnum = getCharNumber(str[i]);
        numout += charnum * Math.pow(26, al - i - 1);
      }
      return numout;
    },
    getTableParams(wb, sheetName) {
      let data = XLSX.utils.sheet_to_json(wb.Sheets[sheetName], {
        header: "A",
      });
      // let csv = XLSX.utils.sheet_to_csv(wb.Sheets[sheetName], {FS:"|"}) || "";
      // console.log(data);
      let max = 0;
      data.forEach((item) => {
        _.forEach(item, (val, key) => {
          let num = this.str2Num(key);
          if (num > max) {
            max = num;
          }
        });
      });
      let columns = Array(max)
        .fill("")
        .map((item, index) => {
          let key = this.num2Str(index + 1);
          return {
            name: key,
            prop: key,
            width: 200,
          };
        });
      columns.unshift({ prop: "_index", name: "#", width: 80 });
      return {
        config: columns,
        data,
        sourceData: data,
        height: document.body.clientHeight - 260,
        itemHeight: 55,
        minWidth: (columns.length - 1) * 200 + 80,
      };
    },
    onChange(key) {
      const { sheets } = this;
      const index = sheets.findIndex((item) => item.name === key);
      if (_.isEmpty(_.get(sheets, `${index}.tableParams`))) {
        NProgress.set(0.4);
        setTimeout(() => {
          this.$set(
            this.sheets[index],
            `tableParams`,
            this.getTableParams(this.wb, key)
          );
          NProgress.done();
        }, 500);
      }
      this.activeKey = key + "";
    },
    onFilterCompare() {
      this.$refs.Modal.show({
        sheets: this.sheets,
        count: this.count,
      });
    },
    onSave(data = {}) {
      if (data.flag) {
        // 过滤
        const {
          baseSheet,
          baseCol,
          baseVal,
          compareSheets,
          compareCol,
          compareVal,
        } = data;
        let sheets = [baseSheet, ...compareSheets];

        NProgress.set(0.4);
        setTimeout(() => {
          this.sheets = this.sheets.filter((item) =>
            sheets.includes(item.name)
          );
          let arr = [...this.sheets];
          arr.forEach((item, index) => {
            if (_.isPlainObject(_.get(item, "tableParams"))) {
              arr[index].tableParams = this.getTableParams(this.wb, item.name);
            }
          });
          this.sheets = arr;
          this.activeKey = arr[0].name;
          setTimeout(() => {
            this.sheets.forEach((item, index) => {
              if (compareSheets.includes(item.name)) {
                let { sourceData = [] } = item.tableParams;
                let matchItems = sourceData.filter((itm) => {
                  return itm[baseCol] == baseVal;
                });
                this.$set(this.sheets[index].tableParams, "data", matchItems);
              }
            });
            NProgress.done();
          }, 300);
        }, 300);
      } else {
        // 还原
        NProgress.set(0.4);
        setTimeout(() => {
          this.sheets = this.wb.SheetNames.map((item, index) => {
            return {
              name: item,
              tableParams: index == 0 ? this.getTableParams(this.wb, item) : {},
            };
          });
          this.activeKey = _.get(this.sheets, "0.name");
          NProgress.done();
        }, 300);
      }
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="less" scoped>
.test-excel {
  padding: 24px;
  position: relative;

  .upload-wrapper {
  }

  /deep/ .item-cell-inner {
    > span {
      overflow: hidden;
      text-overflow: ellipsis;
      display: -webkit-box;
      -webkit-line-clamp: 2;
      -webkit-box-orient: vertical;
    }
  }

  .btn-filter-compare {
    position: absolute;
    right: 24px;
    top: 24px;
  }
}
</style>
