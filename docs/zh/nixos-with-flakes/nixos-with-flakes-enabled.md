# 使用 Flakes 来管理你的 NixOS

## 启用 NixOS 的 Flakes 支持 {#enable-nix-flakes}

与 NixOS 默认的配置方式相比，Nix Flakes 提供了更好的可复现性，同时它清晰的包结构定义原生支持了以其他 Git 仓库为依赖，便于代码分享，因此更建议使用 Nix Flakes 来管理系统配置。

但是目前 Nix Flakes 作为一个实验性的功能，仍未被默认启用。所以我们需要手动启用它，修改 `/etc/nixos/configuration.nix` 文件，在函数块中启用 flakes 与 nix-command 功能：

```nix{15,18-19}
# Edit this configuration file to define what should be installed on
# your system.  Help is available in the configuration.nix(5) man page
# and in the NixOS manual (accessible by running ‘nixos-help’).
{ config, pkgs, ... }:

{
  imports =
    [ # Include the results of the hardware scan.
      ./hardware-configuration.nix
    ];

  # 省略掉前面的配置......

  # 启用 Nix Flakes 功能，以及配套的新 nix-command 命令行工具
  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  environment.systemPackages = with pkgs; [
    git  # Nix Flakes 通过 git 命令从数据源拉取依赖，所以必须先安装好 git
    vim
    wget
    curl
  ];
  # 将默认编辑器设置为 vim
  environment.variables.EDITOR = "vim";

  # 省略其他配置......
}
```

然后运行 `sudo nixos-rebuild switch` 应用修改后，即可使用 Nix Flakes 来管理系统配置。

额外还有个好处就是，现在你可以通过 `nix repl` 打开一个 nix 交互式环境，有兴趣的话，可以使用它复习测试一遍前面学过的所有 Nix 语法。

## 将系统配置切换到 flake.nix {#switch-to-flake-nix}

在启用了 Nix Flakes 特性后，`sudo nixos-rebuild switch` 命令会优先读取 `/etc/nixos/flake.nix` 文件，如果找不到再尝试使用 `/etc/nixos/configuration.nix`。

可以首先使用官方提供的模板来学习 flake 的编写，先查下有哪些模板：

```bash
nix flake show templates
```

其中有个 `templates#full` 模板展示了所有可能的用法，可以看看它的内容：

```bash
nix flake init -t templates#full
cat flake.nix
```

我们参照该模板创建文件 `/etc/nixos/flake.nix` 并编写好配置内容，后续系统的所有修改都将全部由 Nix Flakes 接管，示例内容如下：

```nix
{
  description = "Ryan's NixOS Flake";

  # 这是 flake.nix 的标准格式，inputs 是 flake 的依赖，outputs 是 flake 的输出
  # inputs 中的每一项依赖都会在被拉取、构建后，作为参数传递给 outputs 函数
  inputs = {
    # flake inputs 有很多种引用方式，应用最广泛的格式是:
    #     github:owner/name/reference
    # 即 github 仓库地址 + branch/commit-id/tag

    # NixOS 官方软件源，这里使用 nixos-23.11 分支
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
    # home-manager，用于管理用户配置
    home-manager = {
      url = "github:nix-community/home-manager/release-23.11";
      # `follows` 是 inputs 中的继承语法
      # 这里使 sops-nix 的 `inputs.nixpkgs` 与当前 flake 的
      # `inputs.nixpkgs` 保持一致，避免依赖的 nixpkgs 版本不一致导致问题
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  # outputs 即 flake 的所有输出，其中的 nixosConfigurations 即 NixOS 系统配置
  # flake 有很多用途，也可以有很多不同的 outputs，nixosConfigurations 只是其中一种
  #
  # outputs 是一个函数，它的参数都在 inputs 中定义，可以通过 inputs 中定义的
  # 名称来引用。
  # 比如这里的输入参数 `nixpkgs`，就是上面 inputs 中的 `nixpkgs`
  # 不过 self 是个例外，这个特殊参数指向 outputs 自身（自引用），以及 flake 根目录
  # 这里的 @ 语法将函数的参数 attribute set 取了个别名，方便在内部使用
  outputs = { self, nixpkgs, ... }@inputs: {
    # 名为 nixosConfigurations 的 outputs 会在执行
    # `sudo nixos-rebuild switch` 时被使用
    # 默认情况下上述命令会使用与主机 hostname 同名的 nixosConfigurations
    # 但是也可以通过 `--flake /path/to/flake/direcotry#nixos-test` 来指定
    # 在 flakes 配置文件夹中执行如下命令即可部署此配置：
    #     sudo nixos-rebuild switch --flake .#nixos-test
    # 其中 --flake 后的参数简要说明如下：
    #   1. `.` 表示使用当前文件夹的 Flakes 配置，
    #   2. `#` 后面的内容则是 nixosConfigurations 的名称
    nixosConfigurations = {
      # hostname 为 nixos-test 的主机会使用这个配置
      # 这里使用了 nixpkgs.lib.nixosSystem 函数来构建配置，
      # 后面的 attributes set 是它的参数，在 nixos 系统上使用如下命令即可部署此配置：
      #     nixos-rebuild switch --flake .#nixos-test
      "nixos-test" = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";

        # Nix 模块系统可将配置模块化，提升配置的可维护性
        #
        # modules 中每个参数，都是一个 Nixpkgs Module
        # nixpkgs manual 中有半份介绍它的文档：
        #   <https://nixos.org/manual/nixpkgs/unstable/#module-system-introduction>
        # 说半份是因为它的文档不全，只有一些简单的介绍（Nix 文档现状...）
        #
        # Nixpkgs Module 可以是一个 attribute set，
        # 也可以是一个返回 attribute set 的函数，如果是函数，
        # 那么它的参数就是当前的 NixOS Module 的参数.
        #
        # 根据 Nix Wiki 对 Nix modules 的描述，默认情况下，
        # Nixpkgs Module 函数的默认参数有几个：
        #
        #  lib:     nixpkgs 自带的函数库，提供了许多操作 Nix 表达式的实用函数
        #           详见 https://nixos.org/manual/nixpkgs/stable/#id-1.4
        #  config:  当前 flake 的所有 config 参数的集合，比较常用
        #  options: 当前 flake 中所有 NixOS Modules 中定义的所有参数的集合
        #  pkgs:    一个包含所有 nixpkgs 包的集合，它也提供了许多相关的工具函数
        #           入门阶段可以认为它的默认值为 `nixpkgs.legacyPackages."${system}"`
        #           可通过 `nixpkgs.pkgs` 这个 option 来自定义 pkgs 的值
        #  modulesPath: 默认 nixpkgs 的内置 Modules 文件夹路径，
        #               常用于从 nixpkgs 中导入一些额外的模块
        #               这个参数通常都用不到，我只在制作 iso 镜像时用到过
        #
        # 上述默认参数都由 Nixpkgs 自动生成。而如果你需要将其他非默认参数传递到
        # 子模块，就得使用 specialArgs 手动设定这些参数，
        # 你可以取消注释如下这行来启用该参数：
        #
        # specialArgs = {...};  # 将 inputs 中的参数传入所有子模块
        modules = [
          # 这里导入之前我们使用的 configuration.nix，这样旧的配置文件仍然能生效
          # 注: configuration.nix 本身也是一个 Nixpkgs Module，因此可以直接在这里导入
          ./configuration.nix
        ];
      };
    };
  };
}
```

这里我们定义了一个名为 `nixos-test` 的系统，它的配置文件为 `./configuration.nix`，这个文件就是我们之前的配置文件，这样我们仍然可以沿用旧的配置。

现在执行 `sudo nixos-rebuild switch --flake /etc/nixos#nixos-test` 应用配置，系统应该没有任何变化，因为我们仅仅是切换到了 Nix Flakes，配置内容与之前还是一致的。

上述代码的注释已经非常详细了，这里再着重说明几点： 

1. `lib` `pkgs` `config` 等默认参数都由 Nixpkgs 自动生成，并可被自动注入到子模块，无需在此处额外声明。
1. `specialArgs = {...};` 这里省略了 attribute set 的内容，其中的内容会被通过名称匹配的方式自动注入到子模块中。
    1. 常见用法比如直接写 `specialArgs = inputs;`，这样所有 inputs 中的 flake 数据源就都可以在子模块中使用了。
    2. 如果你不希望把默认参数与 `inputs` 中的数据源混合到一起，可以改用 `specialArgs = {inherit inputs;};`（即 `specialArgs = {inputs = inputs;};`）.

## 通过 Flakes 来管理系统软件 {#manage-system-software-with-flakes}

切换完毕后，我们就可以通过 Flakes 来管理系统了。管系统最常见的需求就是装软件，我们在前面已经见识过如何通过 `environment.systemPackages` 来安装 `pkgs` 中的包，这些包都来自官方的 nixpkgs 仓库。

现在我们学习下如何通过 Flakes 安装其他来源的软件包，这比直接安装 nixpkgs 要灵活很多，最主要的用途是用来使用 Nixpkgs 中还未添加的某个包的新版本。

以 [helix](https://github.com/helix-editor/helix) 编辑器为例，我们首先需要在 `flake.nix` 中添加 helix 这个 inputs 数据源：

```nix{10,20}
{
  description = "NixOS configuration of Ryan Yin";

  # ......

  inputs = {
    # ......

    # helix editor, use the master branch
    helix.url = "github:helix-editor/helix/master";
  };

  outputs = inputs@{ self, nixpkgs, ... }: {
    nixosConfigurations = {
      nixos-test = nixpkgs.lib.nixosSystem {
        system = "x86_64-linux";

        # 将所有 inputs 参数设为所有子模块的特殊参数，
        # 这样就能直接在子模块中使用 inputs 中的 helix 了
        specialArgs = inputs;
        modules = [
          ./configuration.nix
        ];
      };
    };
  };
}
```

接下来在 `configuration.nix` 中就能引用这个 flake input 数据源了：

```nix{3,14-15}
# Nix 会通过名称匹配，
# 自动将 specialArgs 中的 helix 注入到此函数的第三个参数
{ config, pkgs, helix, ... }:

{
  # 省略掉前面的配置......

  environment.systemPackages = with pkgs; [
    git  # Nix Flakes 通过 git 命令从数据源拉取依赖，所以必须先安装好 git
    vim
    wget
    curl

    # 这里从 helix 这个 inputs 数据源安装了 helix 程序
    helix.packages."${pkgs.system}".helix
  ];

  # 省略其他配置......
}
```

改好后再 `sudo nixos-rebuild switch` 部署，就能安装好 Helix 程序了。
这次部署用时会比以往长挺多，主要是 Nix 会需要使用 master 分支的源码编译整个 Helix.

部署完毕后，可直接在终端使用 `hx` 命令测试验证。

> 如果你的系统 Hostname 不是 `nixos-test`，你需要在 `flake.nix` 中修改 `nixosConfigurations` 的名称，或者使用 `--flake /etc/nixos#nixos-test` 来指定配置名称。

另外，如果你只是想尝试一下 Helix 的最新版本，再决定要不要真正地将它安装到系统里，有更简单的办法，一行命令就行：

> 同样，想使用最新版本的话，源码编译基本是免不了的，这会需要一些时间。

```bash
nix run github:helix-editor/helix/master
```

我们会在后面的 [新一代 Nix 命令行工具的使用](/zh/other-usage-of-flakes/the-new-cli.md) 中详细介绍 `nix run` 的用法。

## 使用其他 Flakes 包提供的功能

其实这才是 Flakes 最主要的功能，一个 Flake 可以依赖其他 Flakes，从而使用它们提供的功能——就如同我们在写 TypeScript/Go/Rust 等程序时使用其他 Library 提供的功能一样。

上面使用 Helix 的官方 Flake 中提供的最新版本就是一个例子，其他更多的用例会在后面提到，这里引用几个后面会讲的例子：

- [Getting Started with Home Manager](./start-using-home-manager.md): 这里引入了社区的 Home-Manager 作为依赖项，从而能直接使用该 Flake 提供的功能。
- [Downgrading or Upgrading Packages](./downgrade-or-upgrade-packages.md): 这里引入了不同版本的 Nixpkgs 作为依赖项，从而能很灵活地选用不同版本的 Nixpkgs 中的包。

