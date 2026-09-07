# WebDAV 私有存储 ACL 修复设计

## 问题

CloudPaste 1.9.1 在计算 API 密钥可访问的挂载点时，会先拒绝所有 `is_public != 1` 的存储配置，然后才检查主体存储 ACL。这样即使 WebDAV API 密钥已明确绑定到某个私有存储配置，该授权也无法生效。实际表现为 `PROPFIND` 可以打开虚拟目录，但 `PUT`、`MOVE`、`GET` 和 `DELETE` 在进入具体挂载点后返回 403。

## 目标

允许命中主体存储 ACL 的 API 密钥访问对应私有存储，同时保持匿名访问、未授权 API 密钥和其他存储配置的现有隔离规则。管理员访问规则保持不变。

## 规则

挂载点带有 `storage_config_id` 时：

1. 如果主体存在非空存储 ACL，只有 ACL 中列出的存储配置可以访问，不再要求这些配置为公开状态。
2. 如果主体没有存储 ACL，继续使用现有回退规则，仅允许访问公开存储。
3. 没有 `storage_config_id` 的挂载点继续沿用现有行为。
4. `basic_path` 的父子路径校验保持不变，并在存储授权通过后继续执行。

## 修改范围

只修改 `backend/src/services/apiKeyService.js` 中 `getAccessibleMountsByBasicPath` 的挂载点过滤条件，不调整数据表、R2 配置、WebDAV 协议处理和管理员认证。

## 验证

使用已绑定 `cloudpaste-files` 的 macOS WebDAV 密钥验证：

- `PROPFIND /dav/files/` 返回 207。
- `PUT` 临时文件成功。
- `MOVE` 临时文件到目标文件成功，用于覆盖 Finder 的写入流程。
- `GET` 下载内容与上传内容一致。
- `DELETE` 删除验证文件成功。
- 未绑定该存储的 API 密钥访问具体挂载点仍返回 403。
- `storage_configs.is_public` 保持为 0。

验证产生的临时文件在测试结束后删除。
