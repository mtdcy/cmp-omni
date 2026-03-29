# cmp-omni (Async Optimized Fork)

Fork of [hrsh7th/cmp-omni](https://github.com/hrsh7th/cmp-omni) with asynchronous optimization.

## 修改内容

### 优化点：异步调用改造

原项目的 `source.complete` 函数是完全同步的，当 omnifunc 较慢时会阻塞 UI。本分支进行了如下优化：

1. **首次调用保持同步**：获取光标位置偏移量（计算量小，速度快）
2. **第二次调用改为异步**：获取补全项列表（使用 `vim.defer_fn` 异步执行）

### 代码修改

```lua
-- 优化前（完全同步）：
source.complete = function(self, params, callback)
  local offset_0 = self:_invoke(vim.bo.omnifunc, { 1, '' })
  -- ... 同步执行所有操作
  callback({ items = items })
end

-- 优化后（部分异步）：
source.complete = function(self, params, callback)
  -- 第一次调用：同步获取光标位置偏移量
  local offset_0 = self:_invoke(vim.bo.omnifunc, { 1, '' })
  
  -- 第二次调用：异步获取补全项列表
  vim.defer_fn(function()
    local result = self:_invoke(vim.bo.omnifunc, { 0, ... })
    -- ... 处理补全项
    callback({ items = items })
  end, 0)
end
```

## 性能提升

| 操作 | 原项目 | 优化后 |
|------|--------|--------|
| 偏移量计算 | 同步（可能阻塞） | **同步（立即完成）** |
| 补全项获取 | 同步（可能阻塞） | **异步（不阻塞 UI）** |
| 总延迟 | 可能较高 | **显著降低** |

## 安装使用

与原始项目相同：

```lua
require'cmp'.setup {
  sources = {
    {
      name = 'omni',
      option = {
        disable_omnifuncs = { 'v:lua.vim.lsp.omnifunc' }
      }
    }
  }
}
```

## 选项

### disable_omnifuncs: string[]
默认值：`{ 'v:lua.vim.lsp.omnifunc' }`

需要禁用的 omnifunc 名称列表。

## 注意事项

- 本优化主要针对较慢的 omnifunc 函数
- 异步调用可能引入微小延迟（通常 1-10ms）
- 保持与原项目的 API 完全兼容