<template>
  <div class="login-container">
    <el-form
      ref="loginForm"
      :model="loginForm"
      :rules="loginRules"
      class="login-form"
      auto-complete="on"
      label-position="left"
    >
      <div class="title-container">
        <h3 class="title">安全态势监控登录</h3>
      </div>

      <el-form-item prop="username">
        <span class="svg-container">
          <svg-icon icon-class="user" />
        </span>
        <el-input
          ref="username"
          v-model="loginForm.username"
          placeholder="用户名"
          name="username"
          type="text"
          tabindex="1"
          auto-complete="on"
        />
      </el-form-item>

      <el-tooltip v-model="capsTooltip" content="Caps lock 已开启" placement="right" manual>
        <el-form-item prop="password">
          <span class="svg-container">
            <svg-icon icon-class="password" />
          </span>
          <el-input
            :key="passwordType"
            ref="password"
            v-model="loginForm.password"
            :type="passwordType"
            placeholder="密码"
            name="password"
            tabindex="2"
            auto-complete="on"
            @keyup.native="checkCapslock"
            @blur="capsTooltip = false"
            @keyup.enter.native="handleLogin"
          />
          <span class="show-pwd" @click="showPwd">
            <svg-icon :icon-class="passwordType === 'password' ? 'eye' : 'eye-open'" />
          </span>
        </el-form-item>
      </el-tooltip>

      <el-button :loading="loading" type="primary" style="width: 100%; margin-bottom: 30px" @click.native.prevent="handleLogin">
        登录
      </el-button>
    </el-form>
  </div>
</template>

<script>
import { validUsername } from '@/utils/validate'

export default {
  name: 'Login',
  data() {
    const validateUsername = (rule, value, callback) => {
      if (!validUsername(value)) {
        callback(new Error('请输入正确的用户名'))
      } else {
        callback()
      }
    }
    const validatePassword = (rule, value, callback) => {
      if (value.length < 6) {
        callback(new Error('密码长度至少为 6 位'))
      } else {
        callback()
      }
    }
    return {
      loginForm: {
        username: 'admin',
        password: '111111'
      },
      loginRules: {
        username: [{ required: true, trigger: 'blur', validator: validateUsername }],
        password: [{ required: true, trigger: 'blur', validator: validatePassword }]
      },
      passwordType: 'password',
      loading: false,
      capsTooltip: false,
      redirect: undefined
    }
  },
  watch: {
    $route: {
      handler(route) {
        const query = route.query || {}
        this.redirect = query.redirect
        this.resetForm()
      },
      immediate: true
    }
  },
  mounted() {
    if (this.loginForm.username === '') {
      this.$refs.username.focus()
    } else if (this.loginForm.password === '') {
      this.$refs.password.focus()
    }
  },
  methods: {
    checkCapslock(e) {
      const { shiftKey, key } = e
      this.capsTooltip = key && key.length === 1 && key >= 'A' && key <= 'Z' && !shiftKey
    },
    showPwd() {
      this.passwordType = this.passwordType === 'password' ? '' : 'password'
      this.$nextTick(() => {
        this.$refs.password.focus()
      })
    },
    resetForm() {
      this.loginForm.password = ''
    },
    handleLogin() {
      this.$refs.loginForm.validate((valid) => {
        if (!valid) {
          return false
        }
        this.loading = true
        this.$store.dispatch('user/login', this.loginForm)
          .then(() => {
            this.$router.push({ path: this.redirect || '/' })
            this.loading = false
          })
          .catch(() => {
            this.loading = false
          })
      })
    }
  }
}
</script>

<style lang="scss" scoped>
$bg: #1f2d3d;
$light_gray: #eee;
$cursor: #fff;

.login-container {
  min-height: 100vh;
  width: 100%;
  background: linear-gradient(135deg, #1f2d3d 0%, #2f3f5f 50%, #1f2d3d 100%);
  overflow: hidden;

  .login-form {
    position: relative;
    width: 380px;
    max-width: 100%;
    margin: 120px auto 0;
    padding: 35px 35px 15px;
    border-radius: 8px;
    background: rgba(0, 0, 0, 0.35);
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.35);
    backdrop-filter: blur(6px);
  }

  .title-container {
    position: relative;
    .title {
      margin: 0 auto 30px;
      text-align: center;
      font-size: 22px;
      color: $light_gray;
      font-weight: 400;
      letter-spacing: 2px;
    }
  }

  .svg-container {
    padding: 6px 5px 6px 15px;
    color: $light_gray;
  }

  .show-pwd {
    position: absolute;
    right: 10px;
    top: 7px;
    font-size: 16px;
    color: $light_gray;
    cursor: pointer;
  }
}

::v-deep .el-input {
  display: inline-block;
  height: 47px;
  width: 100%;
  input {
    background: transparent;
    border: 0;
    border-radius: 0;
    padding: 12px 5px 12px 15px;
    color: $light_gray;
    caret-color: $cursor;
    box-shadow: 0 1px 0 rgba(255, 255, 255, 0.2);
  }
}

::v-deep .el-form-item {
  border: 1px solid rgba(255, 255, 255, 0.1);
  background: rgba(0, 0, 0, 0.3);
  border-radius: 6px;
  color: #454545;
  margin-bottom: 22px;
}
</style>
