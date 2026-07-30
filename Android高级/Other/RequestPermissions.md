- 请求权限，注意Android版本适配

权限请求四种状态：

- 同意

- 本次同意，每次询问

- 禁止

- 禁止且不再询问

| 状态        | 授权成功  | shouldShowRequestPermissionRationale |
| --------- | ----- | ------------------------------------ |
| 同意        | True  |                                      |
| 本次同意，每次询问 | True  |                                      |
| 禁止        | False | False                                |
| 禁止且不再询问   | False |                                      |

高亮：需要厂商ROM自定义适配
