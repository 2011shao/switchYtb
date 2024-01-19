
<template>
  <div>
    <a-typography-text
      >YouToBe小助手
      <a-typography-text type="primary" @click="helpVoid">
        查看教程
      </a-typography-text>
    </a-typography-text>
    <a-spin :loading="bit_loading" style="width: 100%" class="m-t-10">
      <div class="grid-one p-all-1 grid-gap-5">
        <a-divider>视频地址</a-divider>
        <SelectField
          title="视频地址"
          v-model="bit_import_dic.origin_filed"
          :typeNumArr="[1, 15]"
          :preSetArr="['视频地址', '地址', '链接']"
          :allFieldDic="bit_import_dic"
        ></SelectField>
        <a-divider>视频信息</a-divider>
        <SelectTableView
          title="视频信息表"
          canAdd
          v-model="export_table_id"
          :allFieldDic="{ comment_table_id, import_table_id }"
        ></SelectTableView>
        <a-checkbox-group v-model="select_video_info_arr">
          <a-space :size="20" wrap>
            <a-checkbox
              v-for="(value, key) in ytb_video_info_dic"
              :key="key"
              :value="key"
              >{{ value }}</a-checkbox
            >
          </a-space>
        </a-checkbox-group>
        <a-divider>视频评论</a-divider>

        <div class="row-start-center">
          <a-typography-text class="flex-shrink labelCss">
            视频评论
          </a-typography-text>
          <a-radio-group v-model="is_comment">
            <a-radio :value="true">需要</a-radio>
            <a-radio :value="false">不需要</a-radio>
          </a-radio-group>
        </div>
        <SelectTableView
          v-if="is_comment"
          title="评论表"
          canAdd
          v-model="comment_table_id"
          :allFieldDic="{ export_table_id, import_table_id }"
        ></SelectTableView>
        <a-checkbox-group v-if="is_comment" v-model="select_video_comment_arr">
          <a-space :size="20" wrap>
            <a-checkbox
              v-for="(value, key) in ytb_video_comment_dic"
              :key="key"
              :value="key"
              >{{ value }}</a-checkbox
            >
          </a-space>
        </a-checkbox-group>

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
import { ref, onMounted, computed, watch, onUnmounted } from "vue";
import { Message } from "@arco-design/web-vue";
import SelectField from "./superView/SelectField.vue";
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
import dayjs from "dayjs";
const buttonLoading = ref(false);
const is_comment = ref(false);
const bit_import_dic = ref({
  origin_filed: "",
});
const ytb_video_info_dic = ref({
  video_slt: "视频缩略图",
  video_title: "视频标题",
  video_play_num: "播放量",
  video_zan_num: "点赞数",
  video_pl_num: "评论数",
  video_like_num: "收藏数",
  video_description: "视频介绍",
  video_channelTitle: "频道名称",
  video_push_time: "发布时间",
});
const select_video_info_arr = ref([]);
const progress = ref(0);
const select_video_comment_arr = ref([]);

const ytb_video_comment_dic = ref({
  date: "评论时间",
  content: "评论内容",
  sender: "评论人",
});
onMounted(() => {
  select_video_info_arr.value = Object.keys(ytb_video_info_dic.value);
  select_video_comment_arr.value = Object.keys(ytb_video_comment_dic.value);
});

// 导出word
async function get_target_filed_id(selectArr, confDic, tableId) {
  let dic = {};
  for (let key of selectArr) {
    dic[key] = confDic[key];
    const fileId = await addBitNewField(confDic[key], tableId);
    dic[key] = fileId;
  }
  return dic;
}
async function exportVoid() {
  progress.value = 0;
  buttonLoading.value = true;
  bit_loading.value = true;
  let target_filed_dic = await get_target_filed_id(
    select_video_info_arr.value,
    ytb_video_info_dic.value,
    export_table_id.value
  );
  let target_comment_filed_dic = {};

  if (is_comment.value) {
    target_comment_filed_dic = await get_target_filed_id(
      select_video_comment_arr.value,
      ytb_video_comment_dic.value,
      comment_table_id.value
    );
  }

  const recordList = await bit_table.getRecordList();
  const view = await bit_table.getActiveView();
  const recordIdList = await view.getVisibleRecordIdList();
  let newDataArr = [];
  let commentArr = [];
  let i = 0;
  for (const record of recordList) {
    if (!recordIdList.includes(record.id)) {
      continue;
    }
    const cell = await record.getCellByField(bit_import_dic.value.origin_filed);
    const value = cell.val;
    if (!value) {
      i++;
      progress.value = parseInt(i / recordIdList.length);
      continue;
    }
    if (Array.isArray(value) && value.length > 0) {
      const resData = await axios
        .get(
          `https://fsgoole.replit.app/?video_url=${
            value[0]["text"]
          }&comment=${is_comment.value ? "true" : "false"}`
        )
        .catch((err) => {
          i++;
          progress.value = parseInt(i / recordIdList.length);
        });
      if (!resData) continue;
      if (resData.data.code == 0) {
        // 视频信息
        const dic = resultMapDic(resData.data.data, target_filed_dic, record);
        newDataArr.push(dic);
        // 视频评论
        if (is_comment.value) {
          commentArr = getCommentArr(
            resData.data.comments,
            target_comment_filed_dic
          );
        }
      }
    }
    i++;
    progress.value = parseInt(i / recordIdList.length);
  }
  if (export_table_id.value == import_table_id.value) {
    await bit_table.setRecords(newDataArr);
  } else {
    await addBitRecord(newDataArr, export_table_id.value);
  }
  if (is_comment.value && commentArr.length > 0) {
    await addBitRecord(commentArr, comment_table_id.value);
  }
  Message.success("解析完成");
  buttonLoading.value = false;
  bit_loading.value = false;
}
// 视频评论
function getCommentArr(commentsArr, target_filed_dic) {
  const arr = [];
  for (let item of commentsArr) {
    let dic = {};
    dic = {
      fields: {
        [target_filed_dic.date]: item["comment_time"],
        [target_filed_dic.sender]: item["author_name"],
        [target_filed_dic.content]: item["comment_text"],
      },
    };
    for (let key in dic.fields) {
      if (key == "undefined") {
        delete dic.fields[key];
      }
    }
    arr.push(dic);
  }

  return arr;
}

// 视频信息
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
        [target_filed_dic.video_push_time]: dayjs(
          snippet["publishedAt"]
        ).format("YYYY-MM-DD HH:mm:ss"),
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
        [target_filed_dic.video_push_time]: dayjs(
          snippet["publishedAt"]
        ).format("YYYY-MM-DD HH:mm:ss"),
      },
    };
  }
  for (let key in dic.fields) {
    if (key == "undefined") {
      delete dic.fields[key];
    }
  }
  return dic;
}

const commitCan = computed(() => {
  if (is_comment.value) {
    if (
      comment_table_id.value &&
      select_video_comment_arr.value.length > 0 &&
      select_video_info_arr.value.length > 0 &&
      export_table_id.value
    ) {
      return true;
    }
  } else {
    if (select_video_info_arr.value.length > 0 && export_table_id.value) {
      return true;
    }
  }

  return false;
});
function helpVoid(params) {
  window.open(
    "https://y35xdslz6g.feishu.cn/docx/EhI1dCS2boR6nQxmu8fcXiBbnZf?from=from_copylink",
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
