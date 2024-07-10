<script>
import {bitable} from '@lark-base-open/js-sdk';
import {onMounted, ref} from 'vue';
import axios from 'axios';
import {ElButton, ElForm, ElFormItem, ElOption, ElSelect,} from 'element-plus';

export default {
    components: {
      ElButton,
      ElForm,
      ElFormItem,
      ElSelect,
      ElOption,
    },
    setup() {
      //appCode
      const appCode = ref('');
      //当前数据表ID
      const formTableData = ref({ tableId: ''});
      //所有的表格信息
      const tableMetaList = ref([]);
      //所选表格的字段信息
      const fieldMetaList=ref({});
      //发票附件链接字段
      const fieldName=ref('');
      //发票抬头字段
      const invoiceTitleField=ref('');
      //纳税人识别号字段
      const taxpayerIdField=ref('');
      //开票日期字段
      const invoiceDateField=ref('');
      //销售方名称字段
      const sellerNameField=ref('');
      //发票金额字段
      const invoiceValueField=ref('');
      const responseData = ref(null);
      // 定义API的基础URL和路径
      const host = "https://dgfp.market.alicloudapi.com";
      const path = "/ocrservice/invoice";
      // 定义请求头，包括Authorization和Content-Type
      const headers = {
        'Authorization': 'APPCODE '+appCode.value,
        'Content-Type': 'application/json; charset=UTF-8'
      }


      //获取当前表格所有字段信息
      const handleTableChange =async () =>{
        const activeTable=await bitable.base.getTable(formTableData.value.tableId);
        fieldMetaList.value=await activeTable.getFieldMetaList();
        // const recordList=await activeTable.getRecords({
        // });
      }


      const addRecord = async () => {
        //获取当前表格对象
        const activeTable=await bitable.base.getTable(formTableData.value.tableId);
        //获取记录ID列表
        const recordIdList = await activeTable.getRecordIdList();
        for (const record of recordIdList) {
          const attachmentField = await activeTable.getField(fieldName.value);
          const attachmentUrls = await attachmentField.getAttachmentUrls(record);
          if (attachmentUrls[0]) {
            try {
              //调用阿里云OCR接口
              const result= await fetchData(attachmentUrls[0]);
              //数据回填
              //纳税人识别号
              if (result['受票方名称'] && invoiceTitleField.value) {
                const res=await activeTable.setCellValue(invoiceTitleField.value, record, result['受票方名称']);
              }
              if (result['受票方税号'] && taxpayerIdField.value) {
                await activeTable.setCellValue(taxpayerIdField.value, record, result['受票方税号']);
              }
              if (result['开票日期'] && invoiceDateField.value) {
                await activeTable.setCellValue(invoiceDateField.value,record,  result['开票日期']);
              }
              if (result['销售方名称'] && sellerNameField.value) {
                await activeTable.setCellValue(sellerNameField.value,record,  result['销售方名称']);
              }
              if (result['发票金额'] && invoiceValueField.value) {
                await activeTable.setCellValue(invoiceValueField.value,record,  result['发票金额']);
              }
            } catch (error) {
              console.log(error);
            }

          }
        }

      };
      //调用阿里云OCR接口
      const fetchData = async (attachmentUrls) => {
        try {
          // 请求体
          const bodyData = JSON.stringify({ "url": attachmentUrls , "page_no": 1});
          // 使用axios发送POST请求
          const response = await axios.post(`${host}${path}`, bodyData, { headers });
          // 更新响应数据和状态
          return  response.data.data;
        } catch (error) {
          console.error('Error fetching data:', error);
        }
      };

      onMounted(async () => {
        const [tableList,selection] = await Promise.all([
            //获取所有tableID和tableName
            bitable.base.getTableMetaList(),
            //获取当前tableID
            bitable.base.getSelection()


            //获取选中的字段信息
        ]);
        formTableData.value.tableId = selection.tableId;
        tableMetaList.value = tableList;
        // console.log(tableMetaList);


        // const [tableList, selection,fieldList] = await Promise.all([bitable.base.getTableMetaList(), bitable.base.getSelection()]);
        // formData.value.table = selection.tableId;
        // tableMetaList.value = tableList;
        // fieldList;
      });

      //将name转换成ID
      const getFieldIdByName = async (tableId, fieldName) => {
        // console.log("fieldName:"+fieldName)
        const table = await bitable.base.getTable(tableId);
        const fieldList = await table.getFieldMetaList();
        for (const field of fieldList) {
          // console.log(field.name);
          if (field.name === fieldName) {
            return field.id;
          }
        }
        return null;
      };

      return {
        appCode,
        formTableData,
        tableMetaList,
        fieldMetaList,
        fieldName,
        taxpayerIdField,
        invoiceDateField,
        sellerNameField,
        invoiceTitleField,
        invoiceValueField,
        handleTableChange,
        addRecord,
      };
    },
  };
</script>

<template>
  <el-link href="https://wavqa5ycexs.feishu.cn/docx/OoAxdJzpzoxzcnxg23Rc2WqnnCf" target="_blank" style="display: block; text-align: center; font-weight: bold;">{{$t("howToGetAppCode")}}</el-link>
  <el-input :placeholder="$t('howToGetAppCode')" style="width: 100%" v-model="appCode"></el-input>
  <el-form ref="form" class="form" :model="formTableData" label-position="top">
    <el-form-item :label="$t('form.selectDataTable')" size="large">
        <el-select v-model="formTableData.tableId" :placeholder="$t('form.selectDataTable')" style="width: 100%" @visibleChange="handleTableChange">
          <el-option
            v-for="meta in tableMetaList"
            :key="meta.id"
            :label="meta.name"
            :value="meta.id"
          />
        </el-select>
    </el-form-item>
    <el-form-item :label="$t('form.SelectAttachmentField')" size="large">
      <el-select v-model="fieldName" :placeholder="$t('form.SelectAttachmentField')" style="width: 100%">
        <el-option
            v-for="meta in fieldMetaList"
            :key="meta.id"
            :label="meta.name"
            :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('form.SelectInvoiceHeaderField')" size="large">
      <el-select v-model="invoiceTitleField" :placeholder="$t('form.SelectInvoiceHeaderField')" style="width: 100%">
        <el-option
            v-for="meta in fieldMetaList"
            :key="meta.id"
            :label="meta.name"
            :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('form.SelectTaxpayerIdField')" size="large">
      <el-select v-model="taxpayerIdField" :placeholder="$t('form.SelectTaxpayerIdField')" style="width: 100%">
        <el-option
            v-for="meta in fieldMetaList"
            :key="meta.id"
            :label="meta.name"
            :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('form.SelectInvoiceDateField')" size="large">
      <el-select v-model="invoiceDateField" :placeholder="$t('form.SelectInvoiceDateField')" style="width: 100%">
        <el-option
            v-for="meta in fieldMetaList"
            :key="meta.id"
            :label="meta.name"
            :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('form.SelectSellerNameField')" size="large">
      <el-select v-model="sellerNameField" :placeholder="$t('form.SelectSellerNameField')" style="width: 100%">
        <el-option
            v-for="meta in fieldMetaList"
            :key="meta.id"
            :label="meta.name"
            :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <el-form-item :label="$t('form.SelectInvoiceValueField')" size="large">
      <el-select v-model="invoiceValueField" :placeholder="$t('form.SelectInvoiceValueField')" style="width: 100%">
        <el-option
            v-for="meta in fieldMetaList"
            :key="meta.id"
            :label="meta.name"
            :value="meta.id"
        />
      </el-select>
    </el-form-item>
    <el-button type="primary" plain size="large" @click="addRecord">{{ $t("recognition") }}</el-button>
  </el-form>
</template>

<style scoped>
  .form :deep(.el-form-item__label) {
    font-size: 16px;
    color: var(--el-text-color-primary);
    margin-bottom: 0;
  }
  .form :deep(.el-form-item__content) {
    font-size: 16px;
  }
</style>
