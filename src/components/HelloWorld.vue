<template>
  <el-row :gutter="20">
    <!-- Left column: Command result display -->
    <el-col :xs="24" :sm="16" :lg="18">
      <div class="result-box">
        <el-input
            ref="resultBox"
            type="textarea"
            v-model="resultArea"
            :rows="30"
            resize="none"
            readonly
            class="terminal-output"
        ></el-input>
      </div>
    </el-col>

    <!-- Right column: Command input -->
    <el-col :xs="24" :sm="8" :lg="6">
      <div class="input-box">
        <el-form @submit.prevent @keydown.enter="onKeydownEnter">
          <el-form-item>
            <el-input v-model="inputCommand" placeholder="Enter command..." class="command-input"></el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="submitCommand">Submit</el-button>
          </el-form-item>
        </el-form>
      </div>
    </el-col>
  </el-row>
</template>

<script setup>
//ref 注入全局 axios 实例，用于请求接口
//nextTick 在 DOM 更新完成后执行回调，用于控制页面的滚动条
//inject 注入依赖，使用 Vue 项目全局的 axios 实例
import {ref, nextTick, inject} from 'vue';

const axios = inject('axios');

//username、password、code 保存用户的登录信息
const username = ref('');
const password = ref('');
const code = ref();

//inputCommand 保存用户输入的命令
//tips 显示当前提示信息
//resultArea 显示在终端中的内容
//isLogin 标识用户是否已登录
//inputPassword 标识当前是否需要用户输入密码
const inputCommand = ref('');
const tips = ref('Linux模拟系统\n账号：');
const resultArea = ref(tips.value);
const isLogin = ref(false);
const inputPassword = ref(false);

// 确认删除的标记和路径
const confirmDelete = ref(false);
const deletePath = ref('');

// 提交指令函数
function submitCommand() {
  resultArea.value += inputCommand.value + '\n';

  if (isLogin.value) {
    handleLoggedInCommands();
  } else {
    handleLoginCommands();
  }
}

function handleLoggedInCommands() {
  // 清屏
  if (inputCommand.value === 'clear') {
    clearScreen();
  } else {
    if (confirmDelete.value) {
      handleDeleteConfirmation();
    } else {
      executeCommand(inputCommand.value);
    }
  }
}

function clearScreen() {
  resultArea.value = tips.value;
  inputCommand.value = '';
}

function handleDeleteConfirmation() {
  const validYesResponses = ['yes', 'y'];
  const validNoResponses = ['no', 'n'];

  if (validYesResponses.includes(inputCommand.value.toLowerCase())) {
    inputCommand.value = '';
    axios
        .post('/api/disk', {
          code: code.value,
          input: 'directremove ' + deletePath.value,
        })
        .then(handleResponse)
        .catch(err => {
          resultArea.value += '删除失败: ' + err.message + '\n';
          scrollToBottom();
          confirmDelete.value = false; // Reset confirmDelete on error
        });
  } else if (validNoResponses.includes(inputCommand.value.toLowerCase())) {
    resultArea.value += '取消删除\n\n' + tips.value;
    scrollToBottom();
    inputCommand.value = '';
    confirmDelete.value = false; // Exit confirmation
  } else {
    // Input error handling
    resultArea.value += '输入错误，请重新输入[yes(y)/no(n)]\n';
    scrollToBottom();  // Scroll to the bottom for visibility
    inputCommand.value = '';  // Clear the input for new entry
  }
}
function handleResponse(res) {
  resultArea.value += res.data.msg + '\n';

  if (res.data.status === 15) {
    handleLogout();
  }

  if (res.data.status === 12) {
    confirmDelete.value = true; // If the command requires confirmation
  } else {
    confirmDelete.value = false; // Reset confirmation after handling response
  }

  resultArea.value += tips.value;
  scrollToBottom();
}

function executeCommand(command) {
  axios
      .post('/api/disk', {
        code: code.value,
        input: command,
      })
      .then(handleResponse);

  deletePath.value = command.split(' ')[1]; // 存储删除路径
  inputCommand.value = '';
}

function handleLogout() {
  username.value = '';
  password.value = '';
  isLogin.value = false;
  tips.value = '账号：';
}

function handleLoginCommands() {
  if (!inputPassword.value) {
    promptForPassword();
  } else {
    loginUser();
  }
}

function promptForPassword() {
  username.value = inputCommand.value;
  inputPassword.value = true;
  tips.value = '密码：';
  resultArea.value += '\n' + tips.value;
  inputCommand.value = '';
  scrollToBottom();
}

function loginUser() {
  password.value = inputCommand.value;
  inputPassword.value = false;

  axios
      .post('/api/disk/login', {
        username: username.value,
        password: password.value,
      })
      .then(handleLoginResponse);
}

function handleLoginResponse(res) {
  if (res.data.status === 1) {
    handleLoginSuccess(res.data.msg);
  } else {
    handleLoginFailure(res.data.msg);
  }
}

function handleLoginSuccess(msg) {
  isLogin.value = true;
  resultArea.value += '登录成功\n';
  tips.value = `${username.value}@localhost:~ # `;
  resultArea.value += '\n' + tips.value;
  code.value = msg;
  inputCommand.value = '';
  scrollToBottom();
}

function handleLoginFailure(message) {
  resultArea.value += message + '\n';
  tips.value = '账号：';
  resultArea.value += '\n' + tips.value;
  inputCommand.value = '';
  inputPassword.value = false;
  scrollToBottom();
}

function onKeydownEnter(event) {
  if (event.key === 'Enter') {
    submitCommand();
  }
}

function scrollToBottom() {
  nextTick(() => {
    const resultBox = document.querySelector('.terminal-output textarea');
    resultBox.scrollTop = resultBox.scrollHeight;
  });
}


</script>

<style scoped>
.result-box {
  padding: 10px;
  background-color: #282c34;
  border-radius: 8px;
}

.terminal-output {
  background-color: #1e1e1e;
  color: #00ff00;
  font-family: monospace;
  width: 100%;
}

.input-box {
  margin-top: 10px;
  padding: 20px;
}

.command-input {
  background-color: #fff;
}
</style>