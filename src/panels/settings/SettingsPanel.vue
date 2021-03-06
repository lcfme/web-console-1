<template>
  <div v-if="value" class="settings-panel">
    <div class="toolbar">
      <VIcon @click="onClickClose" name="close" style="width: 20px" />
    </div>
    <div class="main">
      <!-- 侧边栏 -->
      <div class="side">
        <div class="title">Settings</div>
        <div
          v-for="(cfg, index) in configs"
          :key="cfg.title"
          @click="activedIndex = index"
          :class="{selected: activedIndex === index}"
          class="item"
          >
          <span>{{cfg.desc}}</span>
        </div>
      </div>
      <!-- 设置选项 -->
      <div class="content">
        <div class="title">
          <span>{{activedConfig.desc}}</span>
        </div>
        <div
          v-for="(setting, index) in activedConfig.items || []"
          :key="index"
          class="setting"
          :class="setting.type"
        >
          <template v-if="setting.type === 'section'">
            <div class="section">{{setting.desc}}</div>
          </template>
          <template v-else-if="setting.type === 'text'">
            <div class="text">{{setting.desc}}</div>
          </template>
          <template v-else-if="setting.type === 'checkbox'">
            <input type="checkbox" v-model="setting.value" :id="index" @change="onSettingsChanged" />
            <label :for="index">&nbsp;{{setting.desc}}</label>
          </template>
          <template v-else-if="setting.type === 'select'">
            <span>{{setting.desc}}</span>
            <select v-model="setting.value" @change="onSettingsChanged">
              <option disabled value="">Select</option>
              <option
                v-for="option in setting.options"
                :key="option.name"
                :value="option.value">
                {{option.text}}
              </option>
            </select>
          </template>
          <template v-else-if="setting.type === 'link'">
            <a :href="setting.src" alt="" target="_bank">{{setting.desc}}</a>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { VIcon } from "@/components";
import { eventBus, Logger, cloneDeep } from "@/utils";
import { pluginManager } from "@/plugins";

const logger = new Logger("[SettingsPanel]");
const KEY_SETTINGS = "web-console:settings";

/**
 * 默认配置，一组配置段，每个配置段包含一个或多个表单配置项
 * name 配置段名称
 * desc 配置段描述
 * items 表单配置项列表
 *   type 表单类型
 *   desc 字段描述
 *   name 字段名
 *   value 字段值
 */
const defaultConfigs = [
  {
    name: "preferences",
    desc: "Preferences",
    items: [
      { type: "section", desc: "Console" },
      {
        type: "checkbox",
        name: "showTimestamps",
        value: false,
        desc: "Show timestamps"
      },
      {
        type: "select",
        name: "maxMsgCount",
        value: 400,
        desc: "Max message count: ",
        options: [
          { text: "200", value: 200 },
          { text: "400", value: 400 },
          { text: "800", value: 800 },
          { text: "1600", value: 1600 },
          { text: "Infinity", value: Number.MAX_VALUE }
        ]
      },
      { type: "section", desc: "Network" },
      {
        type: "checkbox",
        name: "showRequestType",
        value: false,
        desc: "Show request type"
      },
      { type: "section", desc: "Application" },
      {
        type: "checkbox",
        name: "showApplicationToolbar",
        value: false,
        desc: "Always show toolbar"
      }
    ]
  },
  {
    desc: "About",
    items: [
      { type: "section", desc: "Package Name" },
      { type: "link", desc: process.env.VUE_APP_NAME, src: "https://github.com/whinc/web-console" },
      { type: "section", desc: "Version" },
      {
        type: "link",
        desc: process.env.VUE_APP_VERSION,
        src: "https://www.npmjs.com/package/@whinc/web-console/v/" + process.env.VUE_APP_VERSION
      },
      { type: "section", desc: "Build Date" },
      { type: "text", desc: process.env.VUE_APP_DATE },
      { type: "section", desc: "CHANGELOG" },
      { type: "link", desc: "CHANGELOG", src: "https://github.com/whinc/web-console/blob/master/CHANGELOG.md" },
      { type: "section", desc: "Feedback" },
      { type: "link", desc: "New Issue", src: "https://github.com/whinc/web-console/issues/new" }
    ]
  }
];

export default {
  name: "SettingsPanel",
  components: {
    VIcon
  },
  props: {
    // 是否可见，支持 v-model
    value: Boolean
  },
  data() {
    return {
      activedIndex: 0,
      /**
       * settings 的 UI 配置
       */
      configs: cloneDeep(defaultConfigs)
    };
  },
  computed: {
    activedConfig() {
      return this.configs[this.activedIndex];
    }
  },
  beforeMount() {
    // 安装插件的设置项
    this.installPluginSettings();
    // 加载配置
    this.loadSettings(settings => {
      pluginManager.updateSettings(settings);
      eventBus.emit(eventBus.SETTINGS_LOADED, settings || {});
      return;
    });
  },
  methods: {
    installPluginSettings() {
      const pluginSettings = [];
      const plugins = pluginManager.getPlugins();
      plugins.forEach(plugin => {
        if (!Array.isArray(plugin.settings) || plugin.settings.length === 0) return;
        pluginSettings.push({ type: "section", desc: plugin.name }, ...plugin.settings);
      });

      this.configs[0].items.push(...pluginSettings);
    },
    // 通知配置更新
    onSettingsChanged() {
      const settings = this.extractSettings();
      pluginManager.updateSettings(settings);
      eventBus.emit(eventBus.SETTINGS_CHANGE, settings);
      // logger.log('%o --extract--> %o --expand--> %o', this.configs, settings, this.recoverSettings(settings))
      // logger.log("extractSettings:", settings);
    },
    onClickClose() {
      // 是否可见，支持 v-model
      this.$emit("input", false);

      this.saveSettings();
    },
    saveSettings() {
      const settings = this.extractSettings();
      window.localStorage.setItem(KEY_SETTINGS, JSON.stringify(settings));
    },
    loadSettings(cb) {
      const content = window.localStorage.getItem(KEY_SETTINGS);
      if (!content) {
        cb && cb();
        return;
      }
      try {
        const settings = JSON.parse(content);
        this.recoverSettings(settings);
        cb && cb(settings);
      } catch (err) {
        logger.error(err);
        cb && cb();
      }
    },
    /**
     * 从 UI 配置项中提取设置项
     * [
     *  {
     *    setting: [
     *      {
     *        name: 'a1',
     *        value: 'v1'
     *      },
     *      {
     *        name: 'b1',
     *        value: 'v2'
     *      }
     *    ]
     *  }
     * ]
     *
     * 提取设置：
     * {
     *  a1: v1,
     *  b1: v2
     * }
     */
    extractSettings() {
      const settings = {};
      this.configs.forEach(config => {
        config.items.forEach(item => {
          if (item.name) {
            settings[item.name] = item.value;
          }
        });
      });
      return settings;
    },
    // extractSettings 的逆过程：将设置项恢复到 UI 配置项中
    recoverSettings(settings) {
      this.configs.forEach(config => {
        config.items.forEach(item => {
          if (item.name in settings) {
            item.value = settings[item.name];
          }
        });
      });
    }
  }
};
</script>

<style lang="scss" scoped>
@import "../../styles/variables";
$primary-text-color: rgb(48, 57, 66);
$second-text-color: #777;
.settings-panel {
  $gap: 4px;
  position: absolute;
  top: $gap;
  left: $gap;
  bottom: $gap;
  right: $gap;
  background-color: white;
  box-shadow: 0 0 0.61538462em rgba(0, 0, 0, 0.4);
  display: flex;
  flex-direction: column;
  .toolbar {
    display: flex;
    flex-direction: row;
    justify-content: flex-end;
    min-height: 30px;
    padding: 5px;
  }
  .main {
    display: flex;
    flex-direction: row;
    .side {
      flex: 0 0 30vw;
      display: flex;
      flex-direction: column;
      .title {
        font-size: $primary-font-size * 1.5;
        color: $primary-text-color;
        white-space: nowrap;
        padding: 0 0 10px 10px;
      }
      .item {
        font-size: $primary-font-size;
        flex: 0 0 $primary-font-size * 2;
        padding: 4px 5px;
        color: $second-text-color;
        border-left: 6px solid #ffff;
        display: flex;
        align-items: center;
        justify-content: flex-start;
        &.selected {
          color: $primary-text-color;
          border-left: 6px solid #666;
        }
      }
    }
    .content {
      flex: 1 1 auto;
      display: flex;
      flex-direction: column;
      $margin-left: 15px;
      overflow-y: auto;
      .title {
        font-size: $primary-font-size * 1.5;
        color: $primary-text-color;
        white-space: nowrap;
        padding-bottom: 6px;
        margin-bottom: 10px;
        border-bottom: 1px solid #eeeeee;
      }
      .setting {
        margin-bottom: 12px;
        &.section {
          font-size: 110%;
          color: #222;
        }
        &.text {
          padding-left: $margin-left;
        }
        &.checkbox {
          flex: 0 0 auto;
          padding-left: $margin-left;
          display: flex;
          align-items: center;
          input {
            width: $primary-font-size;
            height: $primary-font-size;
          }
        }
        &.select {
          flex: 0 0 auto;
          padding-left: $margin-left;
          display: flex;
          align-items: center;
          select {
            background-color: transparent;
            border: 1px solid rgba(0, 0, 0, 0.2);
            color: #333;
            border-radius: 2px;
            padding: 0 5px;
            margin: 0 0 0 10px;
          }
        }
        &.link {
          padding-left: $margin-left;
        }
      }
    }
  }
}
</style>
