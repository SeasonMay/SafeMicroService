<template>
  <el-breadcrumb class="app-breadcrumb" separator="/">
    <!-- transition-group用于为列表项提供过渡动画效果 -->
    <transition-group name="breadcrumb">
      <!-- 遍历导航项数组，每项使用路径作为key -->
      <el-breadcrumb-item v-for="(item,index) in levelList" :key="item.path">
        <!-- 如果是最后一项或标记为noRedirect，显示为不可点击文本 -->
        <span v-if="item.redirect==='noRedirect'||index==levelList.length-1" class="no-redirect">{{ item.meta.title }}</span>
        <!-- 否则显示为可点击链接 -->
        <a v-else @click.prevent="handleLink(item)">{{ item.meta.title }}</a>
      </el-breadcrumb-item>
    </transition-group>
  </el-breadcrumb>
</template>

<script>
// 导入path-to-regexp库用于处理动态路由参数
import pathToRegexp from 'path-to-regexp'

export default {
  data() {
    return {
      levelList: null // 存储面包屑导航项
    }
  },
  // 监听路由变化，更新面包屑
  watch: {
    $route() {
      this.getBreadcrumb()
    }
  },
  // 组件创建时初始化面包屑
  created() {
    this.getBreadcrumb()
  },
  methods: {
    // 获取面包屑导航数据
    getBreadcrumb() {
      // 筛选出有meta.title的路由作为面包屑项
      let matched = this.$route.matched.filter(item => item.meta && item.meta.title)
      const first = matched[0] // 获取第一个匹配项

      // 如果第一项不是Dashboard，则添加Dashboard作为起始项
      if (!this.isDashboard(first)) {
        matched = [{ path: '/dashboard', meta: { title: 'Dashboard' }}].concat(matched)
      }

      // 进一步筛选，排除meta.breadcrumb为false的项
      this.levelList = matched.filter(item => item.meta && item.meta.title && item.meta.breadcrumb !== false)
    },

    // 判断路由是否为Dashboard
    isDashboard(route) {
      const name = route && route.name
      if (!name) {
        return false
      }
      // 忽略大小写比较路由名称
      return name.trim().toLocaleLowerCase() === 'Dashboard'.toLocaleLowerCase()
    },

    // 编译路径，处理动态路由参数
    pathCompile(path) {
      // 解决动态路由参数问题
      const { params } = this.$route
      // 使用path-to-regexp将路径模板编译成函数，然后传入实际参数
      var toPath = pathToRegexp.compile(path)
      return toPath(params)
    },

    // 处理面包屑项点击
    handleLink(item) {
      const { redirect, path } = item
      // 如果有重定向属性，则跳转到重定向路径
      if (redirect) {
        this.$router.push(redirect)
        return
      }
      // 否则，编译并跳转到实际路径
      this.$router.push(this.pathCompile(path))
    }
  }
}
</script>

<style lang="scss" scoped>
.app-breadcrumb.el-breadcrumb {
  display: inline-block; // 内联块级显示
  font-size: 14px; // 字体大小
  line-height: 50px; // 行高
  margin-left: 8px; // 左边距

  // 不可点击项样式
  .no-redirect {
    color: #97a8be; // 灰色文本
    cursor: text; // 文本光标
  }
}
</style>
