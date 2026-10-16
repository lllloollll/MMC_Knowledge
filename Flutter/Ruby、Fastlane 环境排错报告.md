# Ruby / Fastlane 环境排错报告

> 排查时间：2026-10-06  
> 环境：macOS 14 / Apple Silicon / Xcode 16.2 / Homebrew Ruby / Fastlane  
> 项目：Android 项目，使用 Bundler 管理 Fastlane

---

## 一、问题背景

项目通过 `Gemfile` 管理 Fastlane，执行：

```bash
bundle install
```

时安装依赖失败。

最初错误集中在：

```text
An error occurred while installing nkf (0.3.0)
```

随后发现多个 Ruby Native Extension 均无法正常编译，包括：

- `nkf 0.3.0`
    
- `io-console 0.9.4`
    
- `bigdecimal 4.1.3`
    
- `erb 6.0.7`
    

因此判断问题并非单个 Gem，而是 Ruby 原生扩展编译环境存在问题。

---

# 二、基础环境

最初 Ruby 环境：

```text
Ruby 4.0.7
RubyGems 4.0.20
Bundler 2.6.3
```

Ruby 路径：

```text
/opt/homebrew/opt/ruby/bin/ruby
```

Xcode：

```text
/Applications/Xcode-16.2.app/Contents/Developer
```

机器：

```text
Apple Silicon
arm64
macOS 14
```

---

# 三、第一次问题：RubyGems 源异常

最初 `gem install nkf -v 0.3.0` 时，RubyGems 使用了：

```text
https://gems.ruby-china.com/
```

并出现 SSL / hostname mismatch。

检查：

```bash
gem sources -l
```

之后将 RubyGems 源恢复为官方源：

```bash
gem sources --add https://rubygems.org/
```

确认：

```bash
gem sources -l
```

结果：

```text
https://rubygems.org/
```

此后可以正常访问 RubyGems。

---

# 四、第二次问题：Native Extension 编译失败

RubyGems 源恢复后：

```bash
gem install nkf -v 0.3.0
```

出现：

```text
Building native extensions...

clang: error: unsupported option '-arch' for target ''
make: *** [nkf.o] Error 1
```

之后 `bundle install` 又发现：

```text
bigdecimal 4.1.3
erb 6.0.7
io-console 0.9.4
nkf 0.3.0
```

多个 Native Extension 都失败。

典型错误：

```text
The compiler failed to generate an executable file.
You have to install development tools first.
```

这说明问题已经从单个 Gem 扩展为：

> Ruby Native Extension 编译环境异常。

---

# 五、关键定位：clang 被 Emscripten 劫持

检查：

```bash
which clang
clang --version
xcrun --find clang
```

发现：

```text
which clang
/Users/casdon/emsdk/upstream/bin/clang
```

版本：

```text
clang version 22.0.0git
Target: unknown
```

而 Xcode 官方 clang：

```text
/Applications/Xcode-16.2.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang
```

因此确定：

> 当前 Shell 中的 `clang` 并不是 Xcode 的 Apple Clang，而是 Emscripten SDK 自带的 clang。

Emscripten clang：

```text
/Users/casdon/emsdk/upstream/bin/clang
```

主要用于 Emscripten/WebAssembly 工具链，不适合作为 macOS Ruby Native Extension 的默认 C 编译器。

这也很好地解释了之前：

```text
clang: error: unsupported option '-arch' for target ''
```

以及：

```text
Target: unknown
```

等异常。

---

# 六、修复 clang 环境

将当前 Shell 的 clang 恢复为系统/Xcode clang：

```bash
export PATH="/usr/bin:/bin:/usr/sbin:/sbin:$PATH"
```

检查：

```bash
which clang
clang --version
```

得到：

```text
/usr/bin/clang
```

以及：

```text
Apple clang version 16.0.0
Target: arm64-apple-darwin23.5.0
```

说明已经恢复为正确的 Apple Clang。

---

# 七、期间出现的误操作：Ruby 被切换成系统 Ruby 2.6

执行：

```bash
export PATH="/usr/bin:/bin:/usr/sbin:/sbin:$PATH"
```

虽然成功修复了 clang，但也把 Homebrew Ruby 从 PATH 中挤到了后面。

导致：

```bash
which ruby
```

变成系统 Ruby：

```text
/usr/bin/ruby
```

版本：

```text
Ruby 2.6
```

同时：

```bash
which gem
```

变成：

```text
/usr/bin/gem
```

执行：

```bash
gem install io-console -v 0.9.4
```

因此出现：

```text
Gem::FilePermissionError

You don't have write permissions for the
/Library/Ruby/Gems/2.6.0 directory.
```

这里并不是 Gem 本身的问题，而是误用了 macOS 系统 Ruby。

---

# 八、恢复 Homebrew Ruby

重新将 Homebrew Ruby 放回 PATH：

```bash
export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
```

检查：

```bash
which ruby
ruby -v

which gem
gem -v
```

恢复为：

```text
/opt/homebrew/opt/ruby/bin/ruby
Ruby 4.0.7

/opt/homebrew/opt/ruby/bin/gem
RubyGems 4.0.20
```

同时 clang 保持：

```text
/usr/bin/clang
Apple clang 16.0.0
```

最终形成正确组合：

```text
Ruby
↓
/opt/homebrew/opt/ruby/bin/ruby
↓
Ruby 4.0.7

Native Extension 编译器
↓
/usr/bin/clang
↓
Apple Clang 16
↓
arm64-apple-darwin23
```

---

# 九、验证 Native Extension

在正确的 Ruby + Apple Clang 环境下重新测试。

## 1. io-console

```bash
gem install io-console -v 0.9.4
```

结果：

```text
Building native extensions. This could take a while...

Successfully installed io-console-0.9.4
```

成功。

---

## 2. nkf

正确命令：

```bash
gem install nkf -v 0.3.0
```

成功：

```text
Successfully installed nkf-0.3.0
```

期间曾误执行：

```bash
gem install nkf -f 0.3.0
```

其中：

```text
-f
```

是：

```text
--force
```

因此 RubyGems 将 `0.3.0` 误认为另一个 Gem 名称。

虽然最终提示：

```text
Could not find a valid gem '0.3.0'
```

但前面的：

```text
Successfully installed nkf-0.3.0
```

说明 `nkf` 实际已经安装成功。

---

## 3. bigdecimal

```bash
gem install bigdecimal -v 4.1.3
```

成功：

```text
Successfully installed bigdecimal-4.1.3
```

---

## 4. erb

```bash
gem install erb -v 6.0.7
```

成功：

```text
Successfully installed erb-6.0.7
```

---

# 十、由此确认第一阶段根因

之前：

```text
Ruby 4.0.7
+
Emscripten clang
```

导致：

```text
Native Extension 编译失败
```

修改为：

```text
Ruby 4.0.7
+
Apple Clang 16
+
arm64
+
Xcode 16.2
```

之后：

```text
io-console 0.9.4   ✅
nkf 0.3.0          ✅
bigdecimal 4.1.3   ✅
erb 6.0.7          ✅
```

因此可以确认：

> **本次 Native Extension 编译失败的主要原因是 Emscripten 的 clang 抢占了 macOS 原生开发环境中的默认 clang。**

---

# 十一、第二阶段问题：Bundler 权限错误

Native Extension 全部解决后重新执行：

```bash
bundle install
```

出现：

```text
Permission denied @ rb_sysopen -
/opt/homebrew/lib/ruby/gems/4.0.0/plugins/rdoc_plugin.rb
```

最初怀疑是 Ruby Gem 目录所有权问题。

检查：

```bash
ls -ld /opt/homebrew/lib/ruby
ls -ld /opt/homebrew/lib/ruby/gems
ls -ld /opt/homebrew/lib/ruby/gems/4.0.0
ls -ld /opt/homebrew/lib/ruby/gems/4.0.0/plugins
```

结果均为：

```text
casdon admin
```

并且：

```bash
find /opt/homebrew/lib/ruby/gems/4.0.0 -user root -print
```

没有发现 root 文件。

因此排除：

> root 权限残留。

---

# 十二、最终定位：rdoc_plugin.rb 本身为只读文件

检查 Homebrew Ruby 中的目标文件：

```bash
ls -l /opt/homebrew/Cellar/ruby/4.0.7/lib/ruby/gems/4.0.0/plugins/rdoc_plugin.rb
```

得到：

```text
-r--r--r-- 1 casdon admin ...
```

权限：

```text
0444
```

即：

```text
owner：只读
group：只读
other：只读
```

但是 Bundler/RubyGems 在安装过程中需要处理该 plugin 文件，因此出现：

```text
Permission denied @ rb_sysopen
```

与此同时测试：

```bash
touch /opt/homebrew/lib/ruby/gems/4.0.0/plugins/test_write
```

成功。

说明：

> Gem plugins 目录本身是可写的，真正不可写的是 Homebrew Ruby 中的 `rdoc_plugin.rb` 文件。

---

# 十三、最终修复

给 Homebrew Ruby 的 `rdoc_plugin.rb` 增加 owner 写权限：

```bash
chmod u+w /opt/homebrew/Cellar/ruby/4.0.7/lib/ruby/gems/4.0.0/plugins/rdoc_plugin.rb
```

使权限从：

```text
-r--r--r--
```

变为：

```text
-rw-r--r--
```

随后重新：

```bash
bundle install
```

Bundler 安装成功。

---

# 十四、最终环境

最终推荐环境：

```text
OS:
macOS 14

Architecture:
Apple Silicon / arm64

Xcode:
16.2

Ruby:
4.0.7

Ruby Path:
/opt/homebrew/opt/ruby/bin/ruby

RubyGems:
4.0.20

Bundler:
2.6.3

Clang:
/usr/bin/clang

Apple Clang:
16.0.0

Target:
arm64-apple-darwin23

Gem Source:
https://rubygems.org/
```

---

# 十五、问题根因总结

本次实际上包含了三个独立问题。

## 问题 1：RubyGems 源

错误：

```text
gems.ruby-china.com
SSL hostname mismatch
```

解决：

```bash
gem sources --add https://rubygems.org/
```

---

## 问题 2：Emscripten clang 抢占默认 clang

错误：

```text
clang: error: unsupported option '-arch' for target ''
```

以及：

```text
Target: unknown
```

原因：

```text
which clang
/Users/casdon/emsdk/upstream/bin/clang
```

解决：

```bash
export PATH="/usr/bin:/bin:/usr/sbin:/sbin:$PATH"
```

并恢复 Homebrew Ruby：

```bash
export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
```

最终保持：

```text
Ruby → Homebrew Ruby
clang → Apple Clang
```

---

## 问题 3：Homebrew Ruby 的 rdoc plugin 不可写

错误：

```text
Permission denied @ rb_sysopen
```

原因：

```text
rdoc_plugin.rb
-r--r--r--
```

解决：

```bash
chmod u+w /opt/homebrew/Cellar/ruby/4.0.7/lib/ruby/gems/4.0.0/plugins/rdoc_plugin.rb
```

---

# 十六、以后遇到类似问题的排查顺序

以后如果 Fastlane / Bundler 又出现 Gem 安装失败，建议按照下面顺序排查。

## ① 首先确认 Ruby

```bash
which ruby
ruby -v
```

应该是：

```text
/opt/homebrew/opt/ruby/bin/ruby
```

不要出现：

```text
/usr/bin/ruby
```

因为 macOS 系统 Ruby 可能是旧版本。

---

## ② 确认 Gem

```bash
which gem
gem -v
```

应该是：

```text
/opt/homebrew/opt/ruby/bin/gem
```

---

## ③ 确认 clang

```bash
which clang
clang --version
```

Apple Silicon macOS 原生开发环境应该优先使用 Apple Clang：

```text
/usr/bin/clang
Apple clang
Target: arm64-apple-darwin
```

如果出现：

```text
/Users/casdon/emsdk/upstream/bin/clang
```

需要警惕 Emscripten 环境影响。

---

## ④ 确认 Xcode

```bash
xcrun --find clang
```

应该指向：

```text
XcodeDefault.xctoolchain/usr/bin/clang
```

---

## ⑤ 确认 Gem 源

```bash
gem sources -l
```

推荐：

```text
https://rubygems.org/
```

---

## ⑥ 再执行 Bundler

```bash
bundle install
```

---

## ⑦ 如果出现 Native Extension 编译错误

优先检查：

```bash
which clang
clang --version
ruby -v
```

不要第一时间修改 Gemfile 或降级 Ruby。

---

# 十七、关于 Ruby 3.4

本次排查过程中曾尝试：

```bash
brew install ruby@3.4
```

但是由于当前 macOS 14 环境下 Homebrew 没有提供对应 bottle，触发：

```text
ruby@3.4
    ↓
llvm
    ↓
源码编译 LLVM
```

并进入：

```text
cmake --build build
```

LLVM 还需要 Rust 等依赖。

但最终证明：

> **当前 Fastlane 环境并不需要为了这个问题降级到 Ruby 3.4。**

在修正 clang 后，Ruby 4.0.7 已经能够成功编译：

```text
io-console
nkf
bigdecimal
erb
```

因此可以继续使用 Ruby 4.0.7。

---

# 十八、最终经验

这次问题最重要的经验不是某一个命令，而是：

> **Ruby Gem 的 Native Extension 编译问题，要先看“到底是谁在编译”。**

重点检查：

```bash
which ruby
which gem
which clang
clang --version
xcrun --find clang
```

尤其是 macOS 开发环境中同时存在：

- Xcode
    
- Homebrew
    
- Emscripten
    
- Flutter
    
- Android NDK
    
- 其他 LLVM/Clang
    

时，很容易因为修改 `PATH` 导致工具链互相污染。

理想状态应该是：

```text
Ruby/Fastlane
    ↓
Homebrew Ruby
    ↓
Apple Clang
    ↓
Xcode SDK
```

而不是：

```text
Ruby/Fastlane
    ↓
Emscripten clang
    ↓
WebAssembly Toolchain
```

---

# 十九、Fastlane 最终验证

环境恢复后，建议最终执行：

```bash
bundle exec fastlane --version
```

然后：

```bash
bundle exec fastlane lanes
```

确认 Bundler 能够正常加载项目中的 Fastlane 及插件。

以后项目中运行 Fastlane，优先使用：

```bash
bundle exec fastlane <lane>
```

而不是直接：

```bash
fastlane <lane>
```

这样可以确保使用 `Gemfile.lock` 中锁定的依赖版本。

---

# 二十、一句话总结

本次 Fastlane 安装失败并不是 Fastlane 本身的问题，而是 **macOS 多套 Ruby/Clang 工具链发生冲突**：

```text
Emscripten clang 抢占 → Native Extension 编译失败
        ↓
恢复 Apple Clang
        ↓
Ruby Native Extension 全部编译成功
        ↓
修复 Homebrew Ruby rdoc_plugin.rb 写权限
        ↓
bundle install 成功
        ↓
Fastlane 环境恢复
```