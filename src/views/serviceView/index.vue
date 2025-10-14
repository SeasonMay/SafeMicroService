<template>
  <!-- 登录容器，承载整个登录表单界面 -->
  <div class="login-container">
    <!--
      Element UI表单组件，包含验证规则
      ref: 用于在JS中引用此表单
      :model: 绑定数据模型
      :rules: 绑定验证规则
     -->
    <el-form ref="loginForm" :model="loginForm" :rules="loginRules" class="login-form" auto-complete="on" label-position="left">

      <!-- 标题容器 -->
      <div class="title-container">
        <h3 class="title">Login Form</h3>
      </div>

      <!-- 用户名输入框表单项 -->
      <el-form-item prop="username">
        <!-- 左侧图标容器 -->
        <span class="svg-container">
          <svg-icon icon-class="user" />
        </span>
        <!--
          用户名输入框
          v-model: 双向数据绑定
          tabindex: Tab键顺序
         -->
        <el-input
          ref="username"
          v-model="loginForm.username"
          placeholder="Username"
          name="username"
          type="text"
          tabindex="1"
          auto-complete="on"
        />
      </el-form-item>

      <!-- 密码输入框表单项 -->
      <el-form-item prop="password">
        <span class="svg-container">
          <svg-icon icon-class="password" />
        </span>
        <!--
          密码输入框
          :key: 当密码类型变化时强制重新渲染
          :type: 动态绑定输入类型(密码或文本)
          @keyup.enter.native: 监听回车键按下事件
         -->
        <el-input
          :key="passwordType"
          ref="password"
          v-model="loginForm.password"
          :type="passwordType"
          placeholder="Password"
          name="password"
          tabindex="2"
          auto-complete="on"
          @keyup.enter.native="handleLogin"
        />
        <!--
          显示/隐藏密码按钮
          @click: 点击事件绑定到showPwd方法
         -->
        <span class="show-pwd" @click="showPwd">
          <svg-icon :icon-class="passwordType === 'password' ? 'eye' : 'eye-open'" />
        </span>
      </el-form-item>

      <!--
        登录按钮
        :loading: 绑定加载状态
        @click.native.prevent: 阻止默认事件并绑定点击事件
       -->
      <el-button :loading="loading" type="primary" style="width:100%;margin-bottom:30px;" @click.native.prevent="handleLogin">Login</el-button>

      <!-- 用户名密码提示区 -->
      <div class="tips">
        <span style="margin-right:20px;">username: admin</span>
        <span> password: any</span>
      </div>

    </el-form>
  </div>
</template>

<script>
// 导入用户名验证函数
import { validUsername } from '@/utils/validate'

export default {
  name: 'Login',
  data() {
    // 用户名验证函数
    const validateUsername = (rule, value, callback) => {
      if (!validUsername(value)) {
        callback(new Error('Please enter the correct user name'))
      } else {
        callback()
      }
    }
    // 密码验证函数 - 至少6位
    const validatePassword = (rule, value, callback) => {
      if (value.length < 6) {
        callback(new Error('The password can not be less than 6 digits'))
      } else {
        callback()
      }
    }
    return {
      // 表单数据模型
      loginForm: {
        username: 'admin', // 默认用户名
        password: '111111' // 默认密码
      },
      // 表单验证规则
      loginRules: {
        username: [{ required: true, trigger: 'blur', validator: validateUsername }],
        password: [{ required: true, trigger: 'blur', validator: validatePassword }]
      },
      loading: false, // 加载状态
      passwordType: 'password', // 密码输入框类型
      redirect: undefined // 重定向路径
    }
  },
  // 监听路由变化
  watch: {
    $route: {
      handler: function(route) {
        // 获取URL中的redirect参数
        this.redirect = route.query && route.query.redirect
      },
      immediate: true // 立即执行一次
    }
  },
  methods: {
    // 切换密码显示/隐藏的方法
    showPwd() {
      if (this.passwordType === 'password') {
        this.passwordType = '' // 显示密码
      } else {
        this.passwordType = 'password' // 隐藏密码
      }
      // 在DOM更新后，使密码输入框获得焦点
      this.$nextTick(() => {
        this.$refs.password.focus()
      })
    },
    // 处理登录的方法
    handleLogin() {
      // 表单验证
      this.$refs.loginForm.validate(valid => {
        if (valid) {
          this.loading = true // 显示加载状态
          // 调用Vuex的login action
          this.$store.dispatch('user/login', this.loginForm).then(() => {
            // 登录成功后重定向
            this.$router.push({ path: this.redirect || '/' })
            this.loading = false // 隐藏加载状态
          }).catch(() => {
            this.loading = false // 登录失败也隐藏加载状态
          })
        } else {
          console.log('error submit!!') // 表单验证失败
          return false
        }
      })
    }
  }
}
</script>

<style lang="scss">
/* 修复input背景不协调和光标变色问题 */
/* 详情见https://github.com/PanJiaChen/vue-element-admin/pull/927 */

// 定义颜色变量
$bg:#283443;
$light_gray:#fff;
$cursor: #fff;

// 解决WebKit浏览器下光标颜色问题
@supports (-webkit-mask: none) and (not (cater-color: $cursor)) {
  .login-container .el-input input {
    color: $cursor;
  }
}

/* 重置Element UI的CSS样式 */
.login-container {
  .el-input {
    display: inline-block;
    height: 47px;
    width: 85%;

    input {
      background: transparent; // 透明背景
      border: 0px; // 无边框
      -webkit-appearance: none; // 移除默认样式
      border-radius: 0px; // 无圆角
      padding: 12px 5px 12px 15px;
      color: $light_gray;
      height: 47px;
      caret-color: $cursor; // 光标颜色

      // 修复Chrome浏览器自动填充背景色问题
      &:-webkit-autofill {
        box-shadow: 0 0 0px 1000px $bg inset !important;
        -webkit-text-fill-color: $cursor !important;
      }
    }
  }

  // 表单项样式
  .el-form-item {
    border: 1px solid rgba(255, 255, 255, 0.1);
    background: rgba(0, 0, 0, 0.1);
    border-radius: 5px;
    color: #454545;
  }
}
</style>

<style lang="scss" scoped>
// 定义颜色变量
$bg:#2d3a4b;
$dark_gray:#889aa4;
$light_gray:#eee;

// 登录容器样式 - 作用域仅限于本组件
.login-container {
  min-height: 100%;
  width: 100%;
  background-color: $bg; // 深蓝色背景
  overflow: hidden;

  // 登录表单样式
  .login-form {
    position: relative;
    width: 520px;
    max-width: 100%;
    padding: 160px 35px 0; // 上部留出大量空间
    margin: 0 auto; // 水平居中
    overflow: hidden;
  }

  // 提示文本样式
  .tips {
    font-size: 14px;
    color: #fff;
    margin-bottom: 10px;

    span {
      &:first-of-type {
        margin-right: 16px;
      }
    }
  }

  // 图标容器样式
  .svg-container {
    padding: 6px 5px 6px 15px;
    color: $dark_gray;
    vertical-align: middle;
    width: 30px;
    display: inline-block;
  }

  // 标题容器样式
  .title-container {
    position: relative;

    .title {
      font-size: 26px;
      color: $light_gray;
      margin: 0px auto 40px auto;
      text-align: center;
      font-weight: bold;
    }
  }

  // 显示密码按钮样式
  .show-pwd {
    position: absolute;
    right: 10px;
    top: 7px;
    font-size: 16px;
    color: $dark_gray;
    cursor: pointer; // 鼠标指针样式
    user-select: none; // 禁止文本选择
  }
}
</style>
