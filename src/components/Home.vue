<template>
  <Page class="page" backgroundSpanUnderStatusBar="true" color="#2c303a">
    <ActionBar class="action-bar btn-primary" title="哔哩下载工具"/>
    <StackLayout>
      <Button class="btn btn-primary" @tap="clipbroad">从粘贴板读取</Button>
      <FlexboxLayout justifyContent="center">
        <Button class="btn btn-primary" @tap="selectQuality">{{ GetQuality }}</Button>
        <Button class="btn btn-primary" @tap="selectClient">{{ GetClient }}</Button>
      </FlexboxLayout>
      <ActivityIndicator :busy="loading" />
      <StackLayout v-if="dllist">
        <FlexboxLayout justifyContent="center">
          <Label :text="dllist.title" />
        </FlexboxLayout>
        <ListView v-if="itemlist" for="item in itemlist">
          <v-template>
            <Button class="btn btn-primary" @tap="onButtonTap(item)">{{ itemtitle(item) }}</Button>
          </v-template>
        </ListView>
      </StackLayout>
    </StackLayout>
  </Page>
</template>

<script>
import { getText } from "nativescript-clipboard";
import { requestPermission } from "nativescript-permissions";
import * as platform from "tns-core-modules/platform";
import * as color from "tns-core-modules/color";
import { android as os } from "tns-core-modules/application";
import { getJSON, getString } from "tns-core-modules/http";
import { alert, action } from "tns-core-modules/ui/dialogs";
import { Folder } from "tns-core-modules/file-system";
import md5 from "js-md5";

const ClientName = [
  { type: "tv.danmaku.bili", name: "正式版" },
  { type: "com.bilibili.app.blue", name: "概念版" },
  { type: "com.bilibili.app.in", name: "Play版" }
];

const Quality = [
  {
    quality: "lua.mp4.bb2api.16",
    description: "清晰",
    video_quality: 16,
    q: 1
  },
  {
    quality: "lua.flv720.bb2api.64",
    description: "高清",
    video_quality: 64,
    q: 2
  },
  {
    quality: "lua.flv.bb2api.80",
    description: "超清",
    video_quality: 80,
    q: 3
  },
  {
    quality: "lua.hdflv2.bb2api.bd",
    description: "1080P",
    q: 4,
    video_quality: 112
  }
];

class BilibiliApi {
  constructor(content) {
    this.content = content;
    return { api: this, info: this.GetLinkInfo() };
  }

  GetAppPath(type) {
    let path = android.os.Environment.getExternalStorageDirectory();
    return `${path}/Android/Data/${ClientName[type].type}/download/`;
  }

  GetQuality(qtype) {
    return Quality[qtype];
  }

  GetSourceId() {
    return this.orig_source_id;
  }

  AppSecret() {
    return "560c52ccd288fed045859ed18bffd973";
  }

  GenSign(url, secret) {
    if (typeof url !== "string" || url.length === 0) return;
    let length = url.indexOf("?", 4);
    if (length === -1) return;
    let params = url.substring(length + 1);
    let sort = params.split("&").sort();
    return md5(sort.join("&") + secret);
  }

  VideoInfo() {
    let url = `http://app.bilibili.com/x/view?aid=${
      this.source_id
    }&appkey=1d8b6e7d45233436&build=521000&ts=${new Date().getTime() / 1000}`;
    return getJSON(`${url}&sign=${this.GenSign(url, this.AppSecret())}`);
  }

  BengumiInfo() {
    let url = `https://bangumi.bilibili.com/view/web_api/season?season_id=${
      this.source_id
    }`;
    return getJSON(url);
  }

  EPBengumiInfo() {
    return getString(
      `https://www.bilibili.com/bangumi/play/ep${this.source_id}`
    )
      .then(content => {
        let match = content.match(/ss(\d+)/);
        if (Array.isArray(match)) return match[0].match(/\d{1,9}/)[0];
      })
      .then(source_id => {
        if (!isNaN(source_id)) {
          this.source_id = source_id;
          return this.BengumiInfo();
        }
      });
  }

  GetLinkInfo() {
    if (typeof this.content !== "string") return;
    let source_id = this.content.match(/\d{1,9}/);
    if (!isNaN(source_id)) {
      this.source_id = source_id;
      this.orig_source_id = source_id;
      let promise = this.GetLinkProcesser();
      return promise.then(({ data, result }) => data || result);
    }
  }

  GetLinkProcesser() {
    if (this.content.indexOf("av") !== -1) return this.VideoInfo();
    else if (this.content.indexOf("ep") !== -1) return this.EPBengumiInfo();
    else return this.BengumiInfo();
  }

  GenDownItem(qtype, ctype, item, data) {
    let time = new Date().getTime();
    let quality = this.GetQuality(qtype);
    let entry = data.isEp
      ? {
          season_id: data.season_id,
          title: data.title,
          cover: data.cover,
          type_tag: quality.quality,
          time_create_stamp: time,
          time_update_stamp: time,
          prefered_video_quality: quality.video_quality,
          source: {
            av_id: item.aid,
            cid: item.cid,
            website: "bangumi",
            webvideo_id: ""
          },
          ep: {
            av_id: item.aid,
            danmaku: item.cid,
            cover: item.cover,
            episode_id: item.ep_id,
            index: item.index,
            index_title: item.index_title,
            page: item.page,
            has_alias: false
          }
        }
      : {
          avid: this.GetSourceId()[0],
          title: data.title,
          cover: data.pic,
          type_tag: quality.quality,
          time_create_stamp: time,
          time_update_stamp: time,
          page_data: {
            cid: item.cid,
            from: item.from,
            part: item.part,
            vid: item.vid,
            tid: data.tid,
            page: item.page
          }
        };
    let finalentry = {
      it_completed: false,
      total_bytes: 0,
      downloaded_bytes: 0,
      guessed_total_bytes: 0,
      total_time_milli: 0,
      danmaku_count: 3000
    };
    let downitem = {
      downPath: this.GetAppPath(ctype),
      av_id: this.GetSourceId(),
      episode_id: data.isEp ? item.ep_id : item.page,
      entry_content: Object.assign(finalentry, entry),
      quality: quality.quality,
      index_content: ""
    };
    return downitem;
  }
}

export default {
  data() {
    return {
      client: 0,
      quality: 1,
      loading: false,
      dllist: null
    };
  },
  computed: {
    itemlist() {
      if (this.dllist && typeof this.dllist === "object") {
        if (this.dllist.episodes) this.dllist.isEp = true;
        return this.dllist.pages || this.dllist.episodes;
      }
      return false;
    },
    GetQuality() {
      let quality = Quality[this.quality] || { description: "未选择" };
      return quality.description;
    },
    GetClient() {
      let client = ClientName[this.client] || { name: "未选择" };
      return client.name || "未选择";
    }
  },
  mounted() {
    this.updateBarColor("#2EBCFF");
  },
  methods: {
    updateBarColor(value) {
      try {
        if (value && platform.device.sdkVersion >= "21") {
          let nativeColor = new color.Color(value).android;
          let activity = os.foregroundActivity || os.startActivity;
          if (!activity)
            return setTimeout(this.updateBarColor.bind(this, value), 500);
          activity.getWindow().setStatusBarColor(nativeColor);
        }
      } catch (err) {
        console.log(err);
      }
    },
    selectQuality() {
      action("请选择清晰度", "取消", Quality.map(obj => obj.description)).then(
        result => {
          let item = Quality.filter(obj => obj.description === result);
          if (item.length > 0) this.quality = Quality.indexOf(item[0]);
        }
      );
    },
    selectClient() {
      action("请选择客户端", "取消", ClientName.map(obj => obj.name)).then(
        result => {
          let item = ClientName.filter(obj => obj.name === result);
          if (item.length > 0) this.client = ClientName.indexOf(item[0]);
        }
      );
    },
    clipbroad() {
      if (this.loading) return;
      this.loading = true;
      getText()
        .then(content => {
          ({ api: this.api, info: this.info } = new BilibiliApi(content));
          if (this.info && this.info instanceof Promise)
            this.info.then(data => {
              if (data) {
                if (data.title.indexOf("僅") !== -1)
                  alert("你下载的是地区专供番，可能需要对应地区网络才能下载");
                this.dllist = data;
              } else alert("获取数据失败，请检查链接或视频号是否正确");
              this.loading = false;
            });
        })
        .catch(e => (this.loading = !alert(`获取数据失败：${e}`)));
    },
    onButtonTap(item) {
      requestPermission("android.permission.WRITE_EXTERNAL_STORAGE")
        .then(this.createDownQueue.bind(this, item))
        .catch(e => alert("创建下载队列时出现错误，请求权限被拒绝"));
    },
    createDownQueue(item) {
      try {
        let dlitem = this.api.GenDownItem(
          this.quality,
          this.client,
          item,
          this.dllist
        );
        let mainpath = (this.dllist.isEp ? "s_" : "") + dlitem.av_id;
        let path = `${dlitem.downPath}${mainpath}/${dlitem.episode_id}/`;
        let folder = Folder.fromPath(path);
        let entryjson = JSON.stringify(dlitem.entry_content);
        folder.getFile("entry.json").writeText(entryjson);
        folder.getFile("danmaku.xml");
        folder.getFolder(dlitem.quality).getFile("index.json");
        alert("创建下载下载队列成功");
      } catch (e) {
        console.log(e);
        alert("创建下载队列时出现错误，请允许本程序的文件读写权限后再次尝试");
      }
    },
    itemtitle(item) {
      return (
        (item.index || item.page) +
        "|" +
        (item.part || item.index_title || "无标题")
      );
    }
  }
};
</script>