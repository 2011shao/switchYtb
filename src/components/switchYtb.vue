
<template>
  <div>
    <!-- <a-typography-text
      >免费OCR识别,更多模型解析持续更新中
      <a-typography-text type="primary" @click="helpVoid">
        查看教程
      </a-typography-text>
    </a-typography-text> -->
    <a-spin :loading="bit_loading" style="width: 100%" class="m-t-10">
      <div class="grid-one p-all-1 grid-gap-5">
        <a-divider>视频地址</a-divider>
        <SelectField
          title="链接地址"
          v-model="bit_import_dic.origin_filed"
          :typeNumArr="[1, 15]"
          :preSetArr="['链接地址', '地址']"
          :allFieldDic="bit_import_dic"
        ></SelectField>
        <a-divider>存放位置</a-divider>
        <SelectTableView
          title="选择表"
          canAdd
          v-model="export_table_id"
          :allFieldDic="{ comment_table_id, import_table_id }"
        ></SelectTableView>
        <a-checkbox-group v-model="select_import_arr">
          <a-space :size="20" wrap>
            <a-checkbox
              v-for="(value, key) in ytb_dic"
              :key="key"
              :value="key"
              >{{ value }}</a-checkbox
            >
          </a-space>
        </a-checkbox-group>
        <div class="row-start-center">
          <a-typography-text class="flex-shrink labelCss">
            获取评论
          </a-typography-text>
          <a-radio-group v-model="is_comment">
            <a-radio value="true">需要</a-radio>
            <a-radio value="false">不需要</a-radio>
          </a-radio-group>
        </div>
        <SelectTableView
          v-if="is_comment"
          title="评论表"
          canAdd
          v-model="comment_table_id"
          :allFieldDic="{ export_table_id, import_table_id }"
        ></SelectTableView>

        <a-button
          :loading="buttonLoading"
          :disabled="!commitCan"
          type="primary"
          @click="exportVoid"
          >开始解析</a-button
        >
        <a-progress v-if="buttonLoading" :percent="progress" />
      </div>
    </a-spin>
  </div>
</template>

<script setup >
import { bitable, FieldType, ITable } from "@lark-base-open/js-sdk";

import { ref, onMounted, computed, watch, onUnmounted } from "vue";
import { Message } from "@arco-design/web-vue";
import SelectField from "./superView/SelectField.vue";
import { getBankBin, getAliBankCheck } from "./bankjs/superBank";
import {
  bit_loading,
  bit_table,
  addBitNewField,
  export_table_id,
  import_table_id,
  addBitRecord,
  comment_table_id,
} from "./js/superBase";
import SelectTableView from "./superView/selectTable.vue";

import axios from "axios";
const buttonLoading = ref(false);
const is_comment = ref(false);
const bit_import_dic = ref({
  origin_filed: "",
});
const ytb_dic = ref({
  video_slt: "视频缩略图",
  video_title: "视频标题",
  video_play_num: "播放量",
  video_zan_num: "点赞数",
  video_pl_num: "评论数",
  video_like_num: "收藏数",
  video_description: "视频介绍",
  video_channelTitle: "频道名称",
});
onMounted(() => {
  const dic = {
    name: {
      sex: "nan",
    },
  };
});

const select_import_arr = ref([]);
const progress = ref(0);

// 导出word
async function exportVoid() {
  progress.value = 0;
  buttonLoading.value = true;
  bit_loading.value = true;
  let target_filed_dic = {};

  for (let key of select_import_arr.value) {
    target_filed_dic[key] = ytb_dic[key];
    const fileId = await addBitNewField(ytb_dic[key]);
    target_filed_dic[key] = fileId;
  }
  if (is_comment.value) {
    const fileId = await addBitNewField(
      "youtube_评论表",
      FieldType.DuplexLink,
      { tableId: comment_table_id.value }
    );
  }

  const recordList = await bit_table.getRecordList();
  const view = await bit_table.getActiveView();
  const recordIdList = await view.getVisibleRecordIdList();
  let newDataArr = [];
  let i = 0;
  const attachmentField = await bit_table.getField(
    bit_import_dic.value.origin_filed
  );
  for (const record of recordList) {
    if (!recordIdList.includes(record.id)) {
      continue;
    }
    const cell = await record.getCellByField(bit_import_dic.value.origin_filed);
    const value = cell.val;
    if (!value) {
      continue;
    }
    if (Array.isArray(value) && value.length > 0) {
      console.log("dddd", value["text"]);
      const resData = await axios.get(
        `https://3132a811-1631-41a4-a032-fbd3bd2807f7-00-2bsd5brb3ja5.sisko.replit.dev/?video_url=${value[0]["text"]}`
      );
      if (resData.data.code == 0) {
        // 相同
        const dic = resultMapDic(resData.data.data, target_filed_dic, record);
        newDataArr.push(dic);
      }

      i++;
      progress.value = parseInt(i / recordIdList.length);
    }
  }
  if (export_table_id.value == import_table_id.value) {
    await bit_table.setRecords(newDataArr);
  } else {
    await addBitRecord(newDataArr);
  }
  Message.success("解析完成");
  buttonLoading.value = false;
  bit_loading.value = false;
}

function resultMapDic(data, target_filed_dic, record) {
  const snippet = data["items"][0]["snippet"];
  const statistics = data["items"][0]["statistics"];
  let dic = {};
  // 相同
  if (export_table_id.value == import_table_id.value) {
    dic = {
      recordId: record.id,
      fields: {
        [target_filed_dic.video_slt]: snippet["thumbnails"]["high"]["url"],
        [target_filed_dic.video_title]: snippet["title"],
        [target_filed_dic.video_pl_num]: statistics["commentCount"],
        [target_filed_dic.video_like_num]: statistics["favoriteCount"],
        [target_filed_dic.video_zan_num]: statistics["likeCount"],
        [target_filed_dic.video_play_num]: statistics["viewCount"],
        [target_filed_dic.video_channelTitle]: snippet["channelTitle"],
        [target_filed_dic.video_description]: snippet["description"],
      },
    };
  } else {
    dic = {
      fields: {
        [target_filed_dic.video_slt]: snippet["thumbnails"]["high"]["url"],
        [target_filed_dic.video_title]: snippet["title"],
        [target_filed_dic.video_pl_num]: statistics["commentCount"],
        [target_filed_dic.video_like_num]: statistics["favoriteCount"],
        [target_filed_dic.video_zan_num]: statistics["likeCount"],
        [target_filed_dic.video_play_num]: statistics["viewCount"],
        [target_filed_dic.video_channelTitle]: snippet["channelTitle"],
        [target_filed_dic.video_description]: snippet["description"],
      },
    };
  }
  debugger;
  for (let key in dic.fields) {
    if (key == "undefined") {
      delete dic.fields[key];
    }
  }
  return dic;
}

const commitCan = computed(() => {
  if (bit_import_dic.value.origin_filed) {
    if (select_import_arr.value.length > 0) {
      return true;
    }
  }
  return false;
});
function helpVoid(params) {
  window.open(
    "https://y35xdslz6g.feishu.cn/docx/QB5udpPFgotthpxKRaJcSaTmnOM?from=from_copylink",
    "_blank"
  );
}
</script>
<style>
.labelCss {
  width: 70px;
  text-align: center;
  flex-shrink: 0;
}
</style>
