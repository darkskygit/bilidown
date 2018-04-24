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
import md5 from "js-md5";

class BilibiliApi {
  static appSecret() {
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
    let url = `http://app.bilibili.com/x/view?aid=${source_id}&appkey=1d8b6e7d45233436&build=521000&ts=${new Date().getTime()}`;
    return getJSON(`${url}&sign=${this.GenSign(url, this.appSecret())}`);
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
}

export default {
  data() {
    return {
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
      this.GetLinkInfo("ep183799").then(data => console.log(this.dllist = data));
    },
    GetLinkInfo(content) {
      if (typeof content !== "string") return;
      let source_id = content.match(/\d{1,9}/);
      if (!isNaN(source_id)) {
        let promise = this.Processer(content)(source_id);
        return promise.then(({ data, result }) => data || result);
      }
    },
    Processer(content) {
      return this.GetLinkProcesser(content).bind(BilibiliApi);
    },
    GetLinkProcesser(content) {
      if (content.indexOf("av") !== -1) return BilibiliApi.VideoInfo;
      else if (content.indexOf("ep") !== -1)
        return BilibiliApi.EPBengumiInfo;
      else return BilibiliApi.BengumiInfo;
    },
    itemtitle(item) {
      return (item.index || item.page) + "|" + (item.part || item.index_title);
    },
    onButtonTap(item) {
      console.log(item);
    }
  }
};
</script>