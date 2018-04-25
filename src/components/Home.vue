<template>
  <Page class="page" backgroundSpanUnderStatusBar="true" color="#2c303a">
    <ActionBar class="action-bar btn-primary" title="哔哩下载工具"/>
    <StackLayout>
      <Button class="btn btn-primary" @tap="clipbroad">clipbroad</Button>
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
import * as platform from "tns-core-modules/platform";
import * as color from "tns-core-modules/color";
import { android } from "tns-core-modules/application";
import { getJSON, getString } from "tns-core-modules/http";
import { alert } from "tns-core-modules/ui/dialogs";
import md5 from "js-md5";

class BilibiliApi {
  static GetClientType() {
    return ["tv.danmaku.bili", "com.bilibili.app.blue", "com.bilibili.app.in"];
  }
  static GetClientPackage(type) {
    return this.GetClientType()[type];
  }
  static GetAppPath(type) {
    return `/sdcard/Android/data/${this.GetClientPackage(type)}/`;
  }
  static AppSecret() {
    return "560c52ccd288fed045859ed18bffd973";
  }
  static GenSign(url, secret) {
    if (typeof url !== "string" || url.length === 0) return;
    let length = url.indexOf("?", 4);
    if (length === -1) return;
    let params = url.substring(length + 1);
    let sort = params.split("&").sort();
    return md5(sort.join("&") + secret);
  }
  static VideoInfo(source_id) {
    let url = `http://app.bilibili.com/x/view?aid=${source_id}&appkey=1d8b6e7d45233436&build=521000&ts=${new Date().getTime() /
      1000}`;
    return getJSON(`${url}&sign=${this.GenSign(url, this.AppSecret())}`);
  }
  static BengumiInfo(source_id) {
    let url = `https://bangumi.bilibili.com/view/web_api/season?season_id=${source_id}`;
    return getJSON(url);
  }
  static EPBengumiInfo(source_id) {
    return getString(`https://www.bilibili.com/bangumi/play/ep${source_id}`)
      .then(content => {
        let match = content.match(/ss(\d+)/);
        if (Array.isArray(match)) return match[0].match(/\d{1,9}/)[0];
      })
      .then(source_id => {
        if (!isNaN(source_id)) {
          return this.BengumiInfo(source_id);
        }
      });
  }
  static GetQuality(qtype) {
    const quality = [
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
    return quality[qtype];
  }
  static GenDownItem(qtype, item, data) {
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
          avid: this.source_id,
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
      downPath: BilibiliApi.GetAppPath(0),
      av_id: data.source_id,
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
      dllist: null,
      source_id: null
    };
  },
  computed: {
    itemlist() {
      if (this.dllist && typeof this.dllist === "object") {
        if (this.dllist.episodes) this.dllist.isEp = true;
        if (this.source_id && typeof this.source_id === "object")
          this.dllist.source_id = this.source_id[0];
        return this.dllist.pages || this.dllist.episodes;
      }
      return false;
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
          let activity = android.foregroundActivity || android.startActivity;
          if (!activity)
            return setTimeout(this.updateBarColor.bind(this, value), 500);
          activity.getWindow().setStatusBarColor(nativeColor);
        }
      } catch (err) {
        console.log(err);
      }
    },
    clipbroad() {
      // getText().then(content => {
      //   let info = this.GetLinkInfo("ep183799");
      //   console.log("GetLinkInfo", info)
      // });
      this.GetLinkInfo("ep183799").then(data => {
        if (data) {
          if (data.title.indexOf("僅") !== -1)
            alert("你下载的是地区专供番，暂未支持下载");
          this.dllist = data;
        } else alert("获取数据失败，请检查链接或视频号是否正确");
      });
    },
    GetLinkInfo(content) {
      if (typeof content !== "string") return;
      let source_id = content.match(/\d{1,9}/);
      if (!isNaN(source_id)) {
        let promise = this.Processer(content, source_id);
        return promise.then(({ data, result }) => data || result);
      }
    },
    Processer(content, source_id) {
      this.source_id = source_id;
      return this.GetLinkProcesser(content).call(BilibiliApi, source_id);
    },
    GetLinkProcesser(content) {
      if (content.indexOf("av") !== -1) return BilibiliApi.VideoInfo;
      else if (content.indexOf("ep") !== -1) return BilibiliApi.EPBengumiInfo;
      else return BilibiliApi.BengumiInfo;
    },
    itemtitle(item) {
      return (item.index || item.page) + "|" + (item.part || item.index_title);
    },
    onButtonTap(item) {
      console.log(BilibiliApi.GenDownItem(0, item, this.dllist));
    }
  }
};
</script>