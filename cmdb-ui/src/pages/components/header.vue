<template>
  <Header>
    <div class="menus">
      <Menu mode="horizontal" theme="dark">
        <Submenu v-for="menu in menus" :name="menu.code" :key="menu.code">
          <template slot="title">
            <!-- <Icon size="large" :type="menu.icon" /> -->
            {{ menu.title }}
          </template>
          <router-link
            v-for="submenu in menu.submenus"
            :key="submenu.code"
            :to="submenu.link || ''"
          >
            <MenuItem :name="submenu.code" :disabled="!submenu.link">{{
              submenu.title
            }}</MenuItem>
          </router-link>
        </Submenu>
      </Menu>
    </div>
    <div class="header-right_container">
      <div class="profile">
        <Dropdown>
          <span style="color: white">{{ user }}</span>
          <Icon :size="18" type="md-arrow-dropdown"></Icon>
          <DropdownMenu slot="list">
            <DropdownItem name="logout" to="/wecmdb/logout">
              <a href="/wecmdb/logout" style="width: 100%; display: block">{{
                $t("logout")
              }}</a>
            </DropdownItem>
            <DropdownItem name="changePassword">
              <router-link to="/setting/change-password">修改密码</router-link>
            </DropdownItem>
          </DropdownMenu>
        </Dropdown>
      </div>
      <div class="language">
        <Dropdown>
          <a href="javascript:void(0)">
            <Icon
              size="16"
              type="ios-globe"
              style="margin-right:5px; cursor: pointer"
            />
            {{ currentLanguage }}
            <Icon type="ios-arrow-down"></Icon>
          </a>
          <DropdownMenu slot="list">
            <DropdownItem
              v-for="(item, key) in language"
              :key="item.id"
              :disabled="item === 'English'"
              @click.native="changeLanguage(key)"
              >{{ item }}</DropdownItem
            >
          </DropdownMenu>
        </Dropdown>
      </div>
    </div>
  </Header>
</template>
<script>
import Vue from "vue";
import { getMyMenus } from "@/api/server.js";

import { MENUS } from "../../const/menus.js";

export default {
  data() {
    return {
      user: "",
      currentLanguage: "",
      language: {
        "zh-CN": "简体中文",
        "en-US": "English"
      },
      menus: []
    };
  },
  methods: {
    changeLanguage(key) {
      if (key === "en-US") return;
      Vue.config.lang = key;
      this.currentLanguage = this.language[key];
      localStorage.setItem("lang", key);
    },
    getLocalLang() {
      let currentLangKey = localStorage.getItem("lang") || navigator.language;
      this.currentLanguage = this.language[currentLangKey];
    },
    async getMyMenus() {
      let { status, data, message, user } = await getMyMenus();
      if (status === "OK") {
        this.user = user;
        data.sort((a, b) => a.seqNo - b.seqNo);
        data.forEach(_ => {
          if (!_.parentId) {
            let menuObj = MENUS.find(m => m.code === _.code);
            if (menuObj) {
              this.menus.push({
                title: this.$lang === "zh-CN" ? menuObj.cnName : menuObj.enName,
                id: _.id,
                submenus: [],
                ..._,
                ...menuObj
              });
            }
          }
        });
        data.forEach(_ => {
          if (_.parentId) {
            let menuObj = MENUS.find(m => m.code === _.code);
            if (menuObj) {
              this.menus.forEach(h => {
                if (_.parentId === h.id) {
                  h.submenus.push({
                    title:
                      this.$lang === "zh-CN" ? menuObj.cnName : menuObj.enName,
                    id: _.id,
                    ..._,
                    ...menuObj
                  });
                }
              });
            }
          }
        });

        this.$emit("allMenus", this.menus);
        window.myMenus = this.menus;
      }
    }
  },
  async created() {
    this.getLocalLang();
    this.getMyMenus();
  },
  watch: {
    $lang: function(lang) {
      this.$router.go(0);
    }
  }
};
</script>

<style lang="scss" scoped>
.header {
  display: flex;

  .ivu-layout-header {
    height: 50px;
    line-height: 50px;
  }
  a {
    color: white;
  }

  .menus {
    display: inline-block;
    .ivu-menu-horizontal {
      height: 50px;
      line-height: 50px;

      .ivu-menu-submenu {
        padding: 0 10px;
      }
    }
  }

  .header-right_container {
    float: right;

    .language,
    .profile {
      float: right;
      display: inline-block;
      vertical-align: middle;
      margin-left: 20px;
    }
  }
}
</style>
