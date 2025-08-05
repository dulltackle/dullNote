---
description: 此规则适用于Electron项目的开发规范，技术方案设计文档的编写保证核心代码符合规范，写代码遵守该规范，确保开发质量和效率。
globs:
alwaysApply: false
---
# Electron项目开发规范

## 项目结构规范
- 采用分层架构设计，明确划分为以下层次：
  - `main` 进程：主进程管理，窗口创建，系统API调用
  - `renderer` 进程：渲染进程，UI界面，用户交互
  - `preload` 脚本：安全的上下文桥接，API暴露
  - `services` 层：业务逻辑服务，数据处理
  - `utils` 层：通用工具函数和辅助模块
  - `assets` 层：静态资源文件
- **依赖方向**
  - 严格遵循依赖方向：renderer → preload → main → services → utils
  - 禁止循环依赖
  - 渲染进程不能直接访问Node.js API，必须通过preload脚本
  - 主进程和渲染进程通过IPC通信

## 编码规范
### 命名约定
- **文件命名**
  - 使用大驼峰命名法
  - 例如：`UserService.js`、`MainWindow.js`、`AppConfig.json`
- **变量命名**
  - 使用驼峰命名法
  - 局部变量使用小驼峰（`userId`）
  - 全局变量和类使用大驼峰（`UserService`）
  - 常量使用全大写，下划线分隔（`MAX_RETRY_COUNT`）
- **函数和类命名**
  - 使用驼峰命名法
  - 函数名应该是动词或动词短语
  - 类名应该是名词或名词短语
  - 事件处理函数以 "handle" 或 "on" 开头

### 代码组织
- **模块导入顺序**
  - 首先是Node.js内置模块
  - 然后是第三方依赖
  - 最后是项目内部模块
  - 组之间用空行分隔
```javascript
const { app, BrowserWindow, ipcMain } = require('electron');
const path = require('path');

const express = require('express');
const lodash = require('lodash');

const userService = require('./services/user-service');
const appConfig = require('./config/app-config');
```

- **文件结构**
  - 先声明常量和配置
  - 然后是类和函数定义
  - 最后是模块导出和初始化代码

## 错误处理规范
- **使用统一错误处理机制**
  - 定义标准错误类型和错误码
  - 在主进程和渲染进程中统一错误处理格式
- **错误传播**
  - 在服务层中，使用try-catch包装异步操作
  - 通过IPC传递错误信息时，序列化错误对象
- **错误日志记录**
  - 使用electron-log或winston进行日志记录
  - 在错误发生的源头记录日志，避免重复记录
  - 记录错误时包含足够的上下文信息
```javascript
const log = require('electron-log');

try {
  const result = await userService.getUserData(userId);
  return result;
} catch (error) {
  log.error('Failed to get user data:', error);
  throw new AppError('USER_DATA_ERROR', '获取用户数据失败', error);
}
```

## 进程间通信(IPC)规范
- **使用类型安全的IPC通信**
  - 定义清晰的IPC事件名称和数据结构
  - 使用TypeScript接口定义IPC消息格式
- **安全性考虑**
  - 在preload脚本中暴露有限的API
  - 验证来自渲染进程的所有输入
  - 避免在渲染进程中直接执行系统命令
- **性能优化**
  - 避免频繁的IPC通信
  - 批量处理数据传输
  - 使用异步IPC避免阻塞

## 性能优化规范
- **内存管理**
  - 及时清理不再使用的BrowserWindow实例
  - 避免内存泄漏，正确移除事件监听器
  - 使用WeakMap和WeakSet管理对象引用
- **渲染性能**
  - 使用虚拟滚动处理大量数据
  - 实现懒加载和代码分割
  - 优化CSS和JavaScript资源
- **启动性能**
  - 延迟加载非关键模块
  - 使用预加载优化首屏渲染
  - 缓存应用状态和配置

## 安全规范
- **渲染进程安全**
  - 禁用Node.js集成（nodeIntegration: false）
  - 启用上下文隔离（contextIsolation: true）
  - 使用安全的preload脚本
- **内容安全策略**
  - 实施严格的CSP策略
  - 验证所有外部资源
  - 防止XSS和代码注入攻击
- **数据安全**
  - 加密敏感数据存储
  - 使用安全的通信协议
  - 实现适当的权限控制

## 测试规范
- **单元测试**
  - 使用Vitest进行单元测试
  - 为核心业务逻辑编写测试用例
  - 模拟Electron API进行测试
- **集成测试**
  - 使用Playwright进行端到端测试
  - 测试主进程和渲染进程的交互
  - 验证IPC通信的正确性
- **性能测试**
  - 监控内存使用情况
  - 测试启动时间和响应速度
  - 进行压力测试和稳定性测试

## 项目标准组件使用指南
- **日志记录**
  - 使用electron-log进行日志管理
  - 配置日志级别和输出格式
  - 实现日志轮转和清理策略
- **配置管理**
  - 使用electron-store管理应用配置
  - 区分开发和生产环境配置
  - 支持配置热更新
- **自动更新**
  - 使用electron-updater实现自动更新
  - 配置更新服务器和签名验证
  - 实现增量更新和回滚机制
- **数据存储**
  - 使用SQLite或LevelDB进行本地数据存储
  - 实现数据备份和恢复功能
  - 考虑数据迁移和版本兼容性

## 代码示例
### 主进程示例
```javascript
// main/main-process.js
const { app, BrowserWindow, ipcMain } = require('electron');
const path = require('path');
const log = require('electron-log');
const userService = require('../services/user-service');

class MainProcess {
  constructor() {
    this.mainWindow = null;
    this.setupEventHandlers();
  }

  setupEventHandlers() {
    app.whenReady().then(() => {
      this.createMainWindow();
      this.setupIpcHandlers();
    });

    app.on('window-all-closed', () => {
      if (process.platform !== 'darwin') {
        app.quit();
      }
    });
  }

  createMainWindow() {
    this.mainWindow = new BrowserWindow({
      width: 1200,
      height: 800,
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        preload: path.join(__dirname, '../preload/preload.js')
      }
    });

    this.mainWindow.loadFile('renderer/index.html');
  }

  setupIpcHandlers() {
    ipcMain.handle('get-user-data', async (event, userId) => {
      try {
        const userData = await userService.getUserData(userId);
        return { success: true, data: userData };
      } catch (error) {
        log.error('Failed to get user data:', error);
        return { success: false, error: error.message };
      }
    });
  }
}

module.exports = new MainProcess();
```

### Preload脚本示例
```javascript
// preload/preload.js
const { contextBridge, ipcRenderer } = require('electron');

// 安全地暴露API给渲染进程
contextBridge.exposeInMainWorld('electronAPI', {
  // 用户数据相关API
  getUserData: (userId) => ipcRenderer.invoke('get-user-data', userId),

  // 事件监听
  onUserDataUpdate: (callback) => {
    ipcRenderer.on('user-data-updated', callback);
  },

  // 移除事件监听
  removeAllListeners: (channel) => {
    ipcRenderer.removeAllListeners(channel);
  }
});
```

### 渲染进程示例
```javascript
// renderer/js/user-manager.js
class UserManager {
  constructor() {
    this.currentUser = null;
    this.setupEventListeners();
  }

  setupEventListeners() {
    // 监听用户数据更新
    window.electronAPI.onUserDataUpdate((event, userData) => {
      this.handleUserDataUpdate(userData);
    });
  }

  async loadUserData(userId) {
    try {
      const response = await window.electronAPI.getUserData(userId);

      if (response.success) {
        this.currentUser = response.data;
        this.renderUserInfo();
      } else {
        this.showError('加载用户数据失败: ' + response.error);
      }
    } catch (error) {
      console.error('Error loading user data:', error);
      this.showError('网络错误，请稍后重试');
    }
  }

  renderUserInfo() {
    const userInfoElement = document.getElementById('user-info');
    if (this.currentUser && userInfoElement) {
      userInfoElement.innerHTML = `
        <h2>${this.currentUser.name}</h2>
        <p>ID: ${this.currentUser.id}</p>
        <p>邮箱: ${this.currentUser.email}</p>
      `;
    }
  }

  showError(message) {
    const errorElement = document.getElementById('error-message');
    if (errorElement) {
      errorElement.textContent = message;
      errorElement.style.display = 'block';
    }
  }

  handleUserDataUpdate(userData) {
    this.currentUser = userData;
    this.renderUserInfo();
  }
}

// 初始化用户管理器
const userManager = new UserManager();
```

### 服务层示例
```javascript
// services/user-service.js
const log = require('electron-log');
const { AppError } = require('../utils/errors');
const database = require('../utils/database');

class UserService {
  async getUserData(userId) {
    // 参数验证
    if (!userId || typeof userId !== 'string') {
      throw new AppError('INVALID_PARAM', '用户ID不能为空');
    }

    try {
      // 从数据库查询用户数据
      const userData = await database.query(
        'SELECT * FROM users WHERE id = ?',
        [userId]
      );

      if (!userData) {
        throw new AppError('USER_NOT_FOUND', '用户不存在');
      }

      return {
        id: userData.id,
        name: userData.name,
        email: userData.email,
        createdAt: userData.created_at
      };
    } catch (error) {
      if (error instanceof AppError) {
        throw error;
      }

      log.error('Database query failed:', error);
      throw new AppError('DATABASE_ERROR', '数据库查询失败');
    }
  }

  async updateUserData(userId, updateData) {
    // 参数验证
    if (!userId || !updateData) {
      throw new AppError('INVALID_PARAM', '参数不完整');
    }

    try {
      const result = await database.query(
        'UPDATE users SET name = ?, email = ? WHERE id = ?',
        [updateData.name, updateData.email, userId]
      );

      if (result.affectedRows === 0) {
        throw new AppError('USER_NOT_FOUND', '用户不存在');
      }

      return await this.getUserData(userId);
    } catch (error) {
      if (error instanceof AppError) {
        throw error;
      }

      log.error('Failed to update user data:', error);
      throw new AppError('UPDATE_FAILED', '更新用户数据失败');
    }
  }
}

module.exports = new UserService();
```

## 示例
### 良好实践：安全的IPC通信
```javascript
// preload/secure-api.js
const { contextBridge, ipcRenderer } = require('electron');

// 定义允许的IPC频道
const ALLOWED_CHANNELS = {
  'get-user-data': true,
  'update-user-data': true,
  'delete-user-data': true
};

// 安全的IPC调用包装器
function secureInvoke(channel, ...args) {
  if (!ALLOWED_CHANNELS[channel]) {
    throw new Error(`Unauthorized IPC channel: ${channel}`);
  }
  return ipcRenderer.invoke(channel, ...args);
}

contextBridge.exposeInMainWorld('secureAPI', {
  user: {
    getData: (userId) => secureInvoke('get-user-data', userId),
    updateData: (userId, data) => secureInvoke('update-user-data', userId, data),
    deleteData: (userId) => secureInvoke('delete-user-data', userId)
  }
});
```

### 不良实践：不安全的渲染进程配置
```javascript
// 错误示例：不安全的BrowserWindow配置
const mainWindow = new BrowserWindow({
  width: 1200,
  height: 800,
  webPreferences: {
    nodeIntegration: true,        // 错误：启用了Node.js集成
    contextIsolation: false,      // 错误：禁用了上下文隔离
    enableRemoteModule: true      // 错误：启用了远程模块
  }
});

// 错误：在渲染进程中直接使用Node.js API
const fs = require('fs');
const path = require('path');

// 错误：没有输入验证的IPC处理
ipcMain.handle('execute-command', async (event, command) => {
  // 危险：直接执行用户输入的命令
  const { exec } = require('child_process');
  return new Promise((resolve, reject) => {
    exec(command, (error, stdout, stderr) => {
      if (error) reject(error);
      else resolve(stdout);
    });
  });
});
```