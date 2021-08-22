<template>
  <a-drawer
    title="条件输入"
    width="65%"
    :closable="false"
    :visible="visible"
    @close="onClose"
  >
    <a-form-model
      ref="form"
      :layout="'horizontal'"
      :model="data"
      v-bind="formItemLayout"
      :style="{ paddingBottom: '60px' }"
    >
      <a-form-model-item label="启动比较">
        <a-switch v-model="data.flag"></a-switch>
      </a-form-model-item>
      <a-form-model-item
        label="基准sheet"
        prop="baseSheet"
        :rules="[{ required: true }]"
      >
        <a-select
          v-model="data.baseSheet"
          :options="baseSheetOptions"
          placeholder="请选择"
          :disabled="!data.flag"
          @change="chooseBaseSheet"
        ></a-select>
      </a-form-model-item>
      <a-form-model-item
        label="基准列"
        prop="baseCol"
        :rules="[{ required: true }]"
      >
        <a-select
          v-model="data.baseCol"
          :options="baseColOptions"
          placeholder="请选择"
          :disabled="!data.flag"
          @change="chooseBaseCol"
        />
      </a-form-model-item>
      <a-form-model-item
        label="基准值"
        prop="baseVal"
        :rules="[{ required: true }]"
      >
        <a-select
          v-model="data.baseVal"
          :options="baseValOptions"
          placeholder="请选择"
          :disabled="!data.flag"
          mode="multiple"
        />
      </a-form-model-item>
      <a-divider />
      <a-form-model-item
        label="比较sheets"
        prop="compareSheets"
        :rules="[{ required: true }]"
      >
        <a-select
          v-model="data.compareSheets"
          :options="compareSheetsOptions"
          placeholder="请选择"
          :disabled="!data.flag"
          mode="multiple"
        ></a-select>
      </a-form-model-item>
      <a-form-model-item
        label="比较列"
        prop="compareCol"
        :rules="[{ required: true }]"
      >
        <a-input
          v-model="data.compareCol"
          placeholder="请输入"
          :disabled="!data.flag"
        />
      </a-form-model-item>
      <!-- <a-form-model-item
        label="比较值"
        prop="compareVal"
        :rules="[{ required: true }]"
      >
        <a-input
          v-model="data.compareVal"
          placeholder="请输入"
          :disabled="!data.flag"
        />
      </a-form-model-item> -->
    </a-form-model>
    <div
      :style="{
        position: 'absolute',
        bottom: 0,
        right: 0,
        borderTop: '1px solid #e8e8e8',
        padding: '10px 16px',
        textAlign: 'right',
        left: 0,
        background: '#fff',
        borderRadius: '0 0 4px 4px',
      }"
    >
      <a-button style="margin-right: 8px" @click="onClose">取消</a-button>
      <a-button type="primary" @click="onOk">执行</a-button>
    </div>
  </a-drawer>
</template>

<script>
export default {
  components: {},
  data() {
    this.sheets = [];
    this.tableParams = {};
    this.count = 0;
    return {
      visible: false,
      data: {
        flag: true,
      },
      formItemLayout: {
        labelCol: { span: 4 },
        wrapperCol: { span: 14 },
      },
      baseSheetOptions: [],
      baseColOptions: [],
      baseValOptions: [],
      compareSheetsOptions: [],
    };
  },
  methods: {
    show({ sheets = [], count }) {
      this.baseSheetOptions = sheets.map((item) => {
        return {
          label: item.name,
          value: item.name,
        };
      });
      if (this.count != count) {
        this.data = { flag: true };
        this.tableParams = {};
      }
      this.count = count;
      this.sheets = sheets;
      this.visible = true;
    },
    hide() {
      this.visible = false;
    },
    onClose() {
      this.hide();
    },
    // setInfo() {
    //   this.compareSheetsOptions = this.sheets
    //     .map((item) => {
    //       if (item.name === this.data.baseSheet) return;
    //       return {
    //         label: item.name,
    //         value: item.name,
    //       };
    //     })
    //     .filter((item) => item);
    // },
    async chooseBaseSheet(e) {
      let sheet = this.sheets.find((item) => item.name === e) || {};
      if (!sheet) {
        return;
      }
      let tableParams = sheet.tableParams;
      if (_.isPlainObject(tableParams)) {
        // 加载
        tableParams = await this.$parent.loadSheet(e);
      }
      console.log(tableParams, 565656);
      this.tableParams = tableParams || {};

      const { config } = tableParams || {};
      this.data = Object.assign({}, this.data, {
        baseCol: undefined,
        baseVal: undefined,
        compareSheets: undefined,
      });
      this.baseColOptions = config.map((item) => ({
        label: item.prop,
        value: item.prop,
      }));
      this.baseValOptions = [];
      this.compareSheetsOptions = this.sheets
        .map((item) => {
          if (item.name === e) return;
          return {
            label: item.name,
            value: item.name,
          };
        })
        .filter((item) => item);
      // this.setInfo();
    },
    chooseBaseCol(e) {
      const { data } = this.tableParams;
      this.data = Object.assign({}, this.data, {
        baseVal: undefined,
      });
      this.baseValOptions = data.map((item, index) => ({
        label: index + 1 + "_" + (item[e] || ""),
        value: index,
      }));
    },
    onOk() {
      if (!this.data.flag) {
        this.hide();
        setTimeout(() => {
          this.$emit("save", this.data);
        }, 300);
        return;
      }
      this.$refs.form.validate((valid) => {
        if (valid) {
          console.log(this.data);
          this.hide();
          setTimeout(() => {
            this.$emit("save", this.data);
          }, 300);
        }
      });
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="less" scoped>
</style>
