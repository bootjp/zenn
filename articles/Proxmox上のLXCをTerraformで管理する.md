---
title: "Proxmox上のLXCをTerraformで管理する"
emoji: "book"
type: "tech"
topics:
  - "tech"
  - "terraform"
  - "proxmox"
published: false
published_at: "2023-01-19 22:00"
---

# Proxmox 上の LXC を Terraform で管理する

## なぜやるか

- Kubernetes クラスタ用のノードを可能な限り手軽にプロビジョニングしたい
- 自宅で使っているミドルウェアを IaC（Infrastructure as Code）で管理したい

## Proxmox から API トークンを取得する

Proxmox の Web UI にアクセスし、データセンター → API トークン → 追加をクリック

![スクリーンショット 2023-01-18 20.47.48.png](Proxmox%E4%B8%8A%E3%81%AELXC%E3%82%92Terraform%E3%81%A6%E3%82%99%E7%AE%A1%E7%90%86%E3%81%99%E3%82%8B%20/%25E3%2582%25B9%25E3%2582%25AF%25E3%2583%25AA%25E3%2583%25BC%25E3%2583%25B3%25E3%2582%25B7%25E3%2583%25A7%25E3%2583%2583%25E3%2583%2588_2023-01-18_20.47.48.png)

![スクリーンショット 2023-01-18 20.50.37.png](Proxmox%E4%B8%8A%E3%81%AELXC%E3%82%92Terraform%E3%81%A6%E3%82%99%E7%AE%A1%E7%90%86%E3%81%99%E3%82%8B%20/%25E3%2582%25B9%25E3%2582%25AF%25E3%2583%25AA%25E3%2583%25BC%25E3%2583%25B3%25E3%2582%25B7%25E3%2583%25A7%25E3%2583%2583%25E3%2583%2588_2023-01-18_20.50.37.png)

※`トークンID` は `シークレット` とは異なり、トークン ID のサフィックスになる

ここでは簡単のため、root ユーザーに紐付く API キーを作成する

作成が完了すると、以下のように `トークンID` とシークレットが生成される

![Untitled](Proxmox%E4%B8%8A%E3%81%AELXC%E3%82%92Terraform%E3%81%A6%E3%82%99%E7%AE%A1%E7%90%86%E3%81%99%E3%82%8B%20/Untitled.png)

## Terraform のセットアップ

ターミナルで `terraform` コマンドが実行できる環境を手元に用意する

※ mac os であれば brew install terraform 等

この記事では以下のバージョンを用いる

```bash
$ terraform --version
Terraform v1.3.7
on darwin_arm64
```

ここから先は簡単のために、すべて `main.tf` に記載するが、ベストプラクティスとしては、内容ごとにファイルを分ける方法がとられている。

[Best practices for using Terraform | Google Cloud](https://cloud.google.com/docs/terraform/best-practices-for-terraform#separate-directories)

### `main.tf` を作成する

```bash
terraform {
  required_providers {
    proxmox = {
      source = "Telmate/proxmox"
      version = "2.9.11"
    }
  }
}

provider "proxmox" {
  pm_api_url = "https://192.168.0.40:8006/api2/json"
  # tlsの証明書の検証がが困難な場合は以下をコメントアウトする
  # pm_tls_insecure = true
  pm_api_token_id = "{トークンID}"
  pm_api_token_secret = "{シークレット}"
}
```

`pm_api_url` を設定し必要に応じて認証情報も記載する

この例では簡単のために、 `pm_api_token_id` と `pm_api_token_secret` をハードコーディングしているが、[terraform コマンド実行時に環境変数から読み込むことが可能](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs#creating-the-connection-via-username-and-api-token)

[Terraform Registry](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs#creating-the-connection-via-username-and-api-token)

### Terraform の実行

```bash
$ terraform init
```

terraform init をし、ワークスペースの初期化とプラグインのダウンロードを行う

次に terraform plan を行うことで実行内容の確認ができるが、ここでは実行内容がないため、今行っても以下のような出力となる

```bash
$ terraform plan

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are
needed.
```

### LXC でコンテナをたてる

`main.tf` に以下を追記する

`target_node` と `ostemplate` などは適宜環境に合わせ変更をする

```bash
resource "proxmox_lxc" "basic" {
  target_node  = "pve5amd"
  hostname     = "lxc-basic"
  ostemplate   = "nas:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst"
  password     = "BasicLXCContainer"
  unprivileged = true

  rootfs {
    storage = "local-lvm"
    size    = "8G"
  }

  network {
    name   = "eth0"
    bridge = "vmbr0"
    ip     = "dhcp"
  }
}
```

※ ostemplate の `nas` の部分は実際にテンプレートや VM イメージがあるストレージの名称を入れる。

WebUI では以下と対応している。

![Untitled](Proxmox%E4%B8%8A%E3%81%AELXC%E3%82%92Terraform%E3%81%A6%E3%82%99%E7%AE%A1%E7%90%86%E3%81%99%E3%82%8B%20/Untitled%201.png)

CT テンプレートがない場合は以下のようにダウンロードをする

![スクリーンショット 2023-01-18 22.04.45.png](Proxmox%E4%B8%8A%E3%81%AELXC%E3%82%92Terraform%E3%81%A6%E3%82%99%E7%AE%A1%E7%90%86%E3%81%99%E3%82%8B%20/%25E3%2582%25B9%25E3%2582%25AF%25E3%2583%25AA%25E3%2583%25BC%25E3%2583%25B3%25E3%2582%25B7%25E3%2583%25A7%25E3%2583%2583%25E3%2583%2588_2023-01-18_22.04.45.png)

※ ここでは nas にダウンロードしているが、適宜ダウンロード先は変える

### terraform plan をする

```bash
$ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # proxmox_lxc.basic will be created
  + resource "proxmox_lxc" "basic" {
      + arch         = "amd64"
      + cmode        = "tty"
      + console      = true
      + cores        = 1
      + cpulimit     = 0
      + cpuunits     = 1024
      + hostname     = "lxc-basic"
      + id           = (known after apply)
      + memory       = 512
      + onboot       = false
      + ostemplate   = "nas:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst"
      + ostype       = (known after apply)
      + password     = (sensitive value)
      + protection   = false
      + start        = false
      + swap         = 0
      + target_node  = "pve5amd"
      + tty          = 2
      + unprivileged = true
      + unused       = (known after apply)
      + vmid         = (known after apply)

      + network {
          + bridge = "vmbr0"
          + hwaddr = (known after apply)
          + ip     = "dhcp"
          + name   = "eth0"
          + tag    = (known after apply)
          + trunks = (known after apply)
          + type   = (known after apply)
        }

      + rootfs {
          + size    = "8G"
          + storage = "local-lvm"
          + volume  = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if
you run "terraform apply" now.
```

この内容で LXC を作成し、問題なければ terraform apply を実行する

### terraform apply

```bash
$ terraform apply

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # proxmox_lxc.basic will be created
  + resource "proxmox_lxc" "basic" {
      + arch         = "amd64"
      + cmode        = "tty"
      + console      = true
      + cores        = 1
      + cpulimit     = 0
      + cpuunits     = 1024
      + hostname     = "lxc-basic"
      + id           = (known after apply)
      + memory       = 512
      + onboot       = false
      + ostemplate   = "nas:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst"
      + ostype       = (known after apply)
      + password     = (sensitive value)
      + protection   = false
      + start        = false
      + swap         = 0
      + target_node  = "pve5amd"
      + tty          = 2
      + unprivileged = true
      + unused       = (known after apply)
      + vmid         = (known after apply)

      + network {
          + bridge = "vmbr0"
          + hwaddr = (known after apply)
          + ip     = "dhcp"
          + name   = "eth0"
          + tag    = (known after apply)
          + trunks = (known after apply)
          + type   = (known after apply)
        }

      + rootfs {
          + size    = "8G"
          + storage = "local-lvm"
          + volume  = (known after apply)
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

proxmox_lxc.basic: Creating...
proxmox_lxc.basic: Still creating... [10s elapsed]
proxmox_lxc.basic: Creation complete after 10s [id=pve5amd/lxc/114]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

## おわりに

以上で、Terraform を用いて Proxmox 上の LXC を Terraform で管理する準備ができた。

Terraform で環境を管理できるため、都度 WebUI でいじる必要がなく、手軽に環境を作ったり壊したりが可能になった。
