# Helm Chart Development Handson

## はじめに

このハンズオンは Helm Chart の開発方法を実際に体験して理解することを目的とします。

以下の流れで Helm Chart を作成していきます。

1. Chart の作成
2. パッケージング
3. Chart Repository の公開

## 事前準備

Red Hat OpenShift Dev Spaces (RHOSDS) にアクセスし、割り当てられたユーザ名でログインします。その後、 ターミナルを起動します。

- ワークスペース作成方法 : https://\<RHOSDS URL>/#https://github.com/k-srkw/helm-handson.git にアクセス
- ターミナル起動方法 : 「Terminal」タブ -> 「New Terminal」 -> 「wto」

## Chart の作成

Chart の作成は以下の流れで行うことをおすすめします。

1. Chart の雛形を生成する
2. 適用するマニフェストファイルの作成
3. マニフェストファイルのテンプレート化
4. テスト、デバッグ

### Chart の雛形を生成する

割り当てられたユーザ名を環境変数に設定してください。

```
$ HANDSONUSER=<ユーザ名>
```

Chart を作成するためにはまず最初に `helm create` コマンドで Helm Chart のボイラープレートを作成します。これにより Helm Chart の基本的なディレクトリ構成とファイルが作成されます。

```
$ helm create handson-$HANDSONUSER
Creating handson-user1
```

以下のように Chart の雛形が作成されます。

```
handson-user1
├── Chart.yaml
├── charts
├── templates : テンプレート格納ディレクトリ
│   ├── NOTES.txt : helm install 時に表示されるヘルプテキスト
│   ├── _helpers.tpl : Chart 内のテンプレートで利用する共通のテンプレートヘルパー
│   ├── deployment.yaml : 
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml : helm test コマンドで実行できる Integration Test 相当の処理を行う Pod マニフェスト
└── values.yaml
```

今回は Chart 開発の流れを確認するため以下のコマンドで既存のテンプレートを削除します。

```
$ rm -rf handson-$HANDSONUSER/templates/*
```

### 適用するマニフェストファイルの作成

実際にデプロイする対象のマニフェストファイルを作成します。
今回はシンプルな　 Node.js アプリケーションを起動する Chart を作成します。

```
$ cp nodejs-template/* handson-$HANDSONUSER/templates/
```

作成できたら実際に OpenShift にデプロイできることを確認します。

```
$ oc project $HANDSONUSER-devspaces
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm list
$ oc get deployment nodejs
```

この時点ではテンプレート化は行っておらず、単純に `templates` ディレクトリに格納したマニフェストファイルがそのまま適用されます。

### マニフェストファイルのテンプレート化

#### 組み込みオブジェクト

`templates` 配下に格納したマニフェストファイルを適用できることを確認できたので、このマニフェストファイルをテンプレート化しリソース名や各フィールドをパラメータ化していきます。

例えば Deployment リソース名 `.metadata.name` を Helm リリース名を含む名前にしたい場合、 {{ .Release.Name }} テンプレートディレクティブを使います。

以下のように `{{  }}` で囲んだテンプレートディレクティブにマニフェストの一部を置き換えることで、 Helm がデプロイ時にテンプレートから実際にデプロイするマニフェストファイルを生成します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs # 変更
...
```

先ほどデプロイした Helm リリースに反映します。事前に Chart が YAML スキーマに沿っているか `helm lint` で確認することをお勧めします。

```
$ helm lint ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deployment handson-$HANDSONUSER-nodejs
```

テンプレートディレクティブ内の `Release` はオブジェクトと呼ばれ、 Helm リリースに関する情報を持っています。 `.Release.Name` はリリース名を保持しています。

Helm には `Release` 以外にも以下の複数の組み込みオブジェクトが用意されており、テンプレート内で利用できるようになっています。

- Release : リリースに関する情報
- Values : デフォルトの values.yaml およびユーザ定義の values.yaml からテンプレートに渡される値
- Chart : Chart.yaml 内の定義
- Files : Chart 内のファイルへのアクセス
- Capabilities : Kubernetes、Helm のバージョンや API リソースのバージョンに関する情報
- Template : テンプレートに関する情報

各組み込みオブジェクトの詳細は [公式ドキュメント](https://helm.sh/docs/chart_template_guide/builtin_objects/) に記載があります。

オブジェクトは `.` で区切られた名前空間として階層構造となっており、先頭の `.` がトップレベルの名前空間から参照していることを示しています。

例えば `Release` オブジェクトの場合以下のようなオブジェクトを利用できます。

- `Release.Name`: リリース名
- `Release.Namespace`: リリース対象の Namespace
- `Release.IsUpgrade`: 現在のリリースがアップグレードまたはロールバックの場合 `true`
- `Release.IsInstall`: 現在のリリースが初回インストールの場合 `true`
- `Release.Revision`: リリースのリビジョン
- `Release.Service`: テンプレートをレンダリングしているサービス。通常は `Helm`

#### Values ファイルによるパラメータ化

組み込みオブジェクトの一つである `Values` はデフォルトの values.yaml およびユーザ定義の values.yaml 、もしくは --set オプションから渡される値をテンプレートで利用できるようにします。

`Values` で保持される値は設定方法によって優先順位があり、以下の順序で設定されます(下に行くほど優先度高)。

- デフォルトの values.yaml
- 親 Chart の values.yaml
- ユーザ定義の values.yaml
- --set オプションで設定されたパラメータ

実際にデフォルトの values.yaml を変更し、値が適用されることを確認します。

```
$ cat << EOF > handson-$HANDSONUSER/values.yaml
appKind: api-level-0-example
EOF
```

テンプレートを以下のように変更します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    service: {{ .Values.appKind }}  # 追加
...
```

values.yaml の値が反映されることを確認します。

```
$ helm lint ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deployment handson-$HANDSONUSER-nodejs --show-labels
```

さらに values.yaml を構造化し、新しいキーを追加します。

```yaml
# values.yaml を以下の内容に変更
labels:
  appKind: api-level-0-example
  provider: handson-user
```

これに合わせてテンプレートも変更します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    service: {{ .Values.labels.appKind }}  # 変更
    provider: {{ .Values.labels.provider }}  # 追加
...
```

values.yaml の値が反映されることを確認します。

```
$ helm lint ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deployment handson-$HANDSONUSER-nodejs --show-labels
```

#### テンプレート関数、パイプラインの利用

Helm Chart では `.Values` オブジェクトなどから取得した値を加工したり、 k8s クラスタ内の情報を取得したりするために利用できるテンプレート関数が用意されています。
例えば `quote` 関数を利用することでテンプレートのレンダリング時に値を引用符で囲むことができます。

また、これらの関数はパイプラインで連結できるようになっており、以下は同じ定義となります。

```yaml
# パイプラインで連結しない例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    service: {{ quote .Values.labels.appKind }}
    provider: {{ quote .Values.labels.provider }}
...
```

```yaml
# パイプラインで連結する例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    service: {{ .Values.labels.appKind | quote }}
    provider: {{ .Values.labels.provider | quote }}
...
```

パイプラインの例として `upper` 関数を追加し連携させます。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    service: {{ .Values.labels.appKind | upper | quote }} # 変更
    provider: {{ .Values.labels.provider | quote }} # 変更
...
```

テンプレート関数により値が加工されることを確認します。

```
$ helm lint ./handson-$HANDSONUSER
# 実際の適用前のレンダリング時点のマニフェスト確認
$ helm upgrade --install --dry-run handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deployment handson-$HANDSONUSER-nodejs --show-labels
```

そのほかにも values.yaml に定義がない場合のデフォルト値を設定する `default` 関数や、 k8s クラスタ上のリソースを参照する `lookup` 関数が利用できます。

Helm には 60 以上の関数が用意されています。Helm は [Go テンプレート言語](https://pkg.go.dev/text/template) がベースとなっており、
そのうちのいくつかは、 Go テンプレート言語自体で定義されています。その他のほとんどは [Sprig](https://masterminds.github.io/sprig/) テンプレートライブラリの一部です。

[公式ドキュメント](https://helm.sh/docs/chart_template_guide/function_list/) で利用可能な関数を確認できます。

#### フロー制御

Helm テンプレートでは If / Else による条件分岐が利用できます。
ここで `eq` は値が同一かどうかを比較する関数です。

```yaml
# 値が同一かどうかを比較する例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    service: {{ .Values.labels.appKind | upper | quote }}
    provider: {{ .Values.labels.provider | quote }}
    {{ if eq .Values.labels.provider "handson-user" }}
    handson-environment: "true"
    {{ end }}
...
```

上記をそのまま実行するとレンダリング時に `if` の部分が空行となります。
`{{- ` と中括弧に続けてハイフンを追加することで空行を詰めることができます。
下側を詰めたい場合は ` -}}` とします。

```yaml
# 値が同一かどうかを比較する例 (空行を詰める場合)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    service: {{ .Values.labels.appKind | upper | quote }}
    provider: {{ .Values.labels.provider | quote }}
    {{- if eq .Values.labels.provider "handson-user" }}
    handson-environment: "true"
    {{- end }}
...
```

オブジェクトのスコープの変更には `with` を利用できます。

例えば以下の例の場合、 `with` アクションの範囲内では各フィールドで毎回 `.Values.favorite` と書くことなくオブジェクトを利用できます。

```yaml
# スコープの変更例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    {{- with .Values.labels }}
    service: {{ .appKind | upper | quote }}
    provider: {{ .provider | quote }}
    {{- end }}
...
```

フロー制御の結果がレンダリング時に反映されることを確認します。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    {{- with .Values.labels }} # 追加
    service: {{ .appKind | upper | quote }}
    provider: {{ .provider | quote }}
    {{- if eq .provider "handson-user" }} # 追加
    handson-environment: "true" # 追加
    {{- end }} # 追加
    {{- end }} # 追加
...
```

```
$ helm lint ./handson-$HANDSONUSER
# 実際の適用前のレンダリング時点のマニフェスト確認
$ helm upgrade --install --dry-run handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deployment handson-$HANDSONUSER-nodejs --show-labels
```

`with` アクションによるスコープ変更の注意点として、親オブジェクトより上のスコープのオブジェクトにはアクセスできなくなります。
例えば以下の `.Release.Name` にはアクセスできずレンダリングに失敗します。

```yaml
    {{- with .Values.labels }}
    service: {{ .appKind | upper | quote }}
    provider: {{ .provider | quote }}
    release: {{ .Release.Name }} # スコープ範囲外のためレンダリングに失敗
    {{- end }}
```

これを解決するには　`.Release.Name` の前に `$` をつけるか、変数を利用します。

`$` から始まるオブジェクトはルートスコープからの参照とみなされ、 `with` 内であっても参照が可能になります。

また、以下のように `with` のスコープの外で変数を定義することでも解決できます。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    {{- $relname := .Release.Name -}} # 追加
    {{- with .Values.labels }}
    service: {{ .appKind | upper | quote }}
    provider: {{ .provider | quote }}
    {{- if eq .provider "handson-user" }}
    handson-environment: "true"
    {{- end }}
    release: {{ $relname }} # 追加
    {{- end }}
...
```

フロー制御の結果がレンダリング時に反映されることを確認します。

```
$ helm lint ./handson-$HANDSONUSER
# 実際の適用前のレンダリング時点のマニフェスト確認
$ helm upgrade --install --dry-run handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deployment handson-$HANDSONUSER-nodejs --show-labels
```

`with` の代わりに `range` アクションと変数を利用することで、複数フィールドを定義するときなどに繰り返し処理を記述することもできます。

```yaml
    {{- range $key, $val := .Values.labels }}
    {{ $key }}: {{ $val | quote }}
    {{- end }}
```

#### 名前付きテンプレート (_helpers.tpl)

プレフィクスに `_` がつくファイルはマニフェストを持たないテンプレートファイルとみなされ、各テンプレートの共通処理の定義に利用されます。

以下のように `define` 関数でテンプレート間でグローバルによび出し可能なテンプレートを定義できます。

```yaml
{{- define "mychart.app" -}}
chartname: {{ .Chart.Name }}
chartversion: "{{ .Chart.Version }}"
{{- end -}}
```

呼び出すときには `include` 関数を利用します。 `indent` 関数で挿入されるフィールドのインデントを変更できます。

```yaml
# 共通処理の利用例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    {{- range $key, $val := .Values.labels }}
    {{ $key | lower }}: {{ $val | quote }}
    {{- end }}
{{ include "mychart.app" . | indent 4 }} # 共通処理を記述したテンプレートを呼び出す
```

実際に `_helpers.tpl` ファイルを作成し、レンダリング時に反映されることを確認します。

```
$ cat << EOF > handson-$HANDSONUSER/templates/_helpers.tpl
{{- define "mychart.app" -}}
chartname: {{ .Chart.Name }}
chartversion: "{{ .Chart.Version }}"
{{- end -}}
EOF
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nodejs
  labels:
    app.kubernetes.io/name: nodejs
    app.kubernetes.io/instance: nodejs
    {{- range $key, $val := .Values.labels }} # 変更
    {{ $key | lower }}: {{ $val | quote }} # 変更
    {{- end }} # 変更
{{ include "mychart.app" . | indent 4 }} # 追加
```

```
$ helm lint ./handson-$HANDSONUSER
# 実際の適用前のレンダリング時点のマニフェスト確認
$ helm upgrade --install --dry-run handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deployment handson-$HANDSONUSER-nodejs --show-labels
```

### テスト、デバッグ

Helm Chart の作成時には変更の都度想定通りマニフェストがレンダリングされているか確認することをお勧めします。

- `helm lint <CHART>` : Chart が YAML スキーマに沿っているか確認
- `helm install --dry-run --debug` : レンダリングされるマニフェストの確認
- `helm template --debug` : API Server と通信せずレンダリングされるマニフェストの確認
- `helm get manifest` : デプロイされたマニフェストの確認
- `helm test` : Integration テスト相当のテストを実行する Pod を起動

helm で Integration テスト相当のテストを実行したい場合、以下のような Pod マニフェストファイルを `templates/tests` 配下に用意します。

[Chart Hook](https://helm.sh/docs/topics/charts_hooks/) 機能を利用して動作するため、 `"helm.sh/hook": test` アノテーションを設定します。
これにより `helm test` 実行時のみ Pod が作成されるようになります。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "test.fullname" . }}-test-connection"
  labels:
    {{- include "test.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "test.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

`helm test <リリース名>` を実行することで Pod が起動し、テストを実行します。

実際にテストを実行してみます。 `values.yaml` を以下のように変更します。 (本来は Service のマニフェストもテンプレート化すべきですが、ここでは割愛します)

```yaml
labels:
  appKind: api-level-0-example
  provider: handson-user
service: # 追加
  port: 8080 # 追加
```

テスト用 Pod マニフェストファイルを作成します。

```
$ mkdir handson-$HANDSONUSER/templates/tests
$ cat << EOF > handson-$HANDSONUSER/templates/tests/test-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ .Release.Name }}-nodejs-test-connection"
  labels:
    appkind: "test"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['nodejs:{{ .Values.service.port }}']
  restartPolicy: Never
EOF
```

テストを実行します。

```
$ helm lint ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm test handson-$HANDSONUSER
```

## パッケージング

作成した Helm Chart を Helm リポジトリ上で公開するためには Chart をアーカイブファイルとしてパッケージングする必要があります。`helm package <CHART-PATH>` で Chart をパッケージングします。

```
$ helm package handson-$HANDSONUSER
# パッケージングされた Helm Chart の内容確認
$ tar tf handson-opentlc-mgr-0.1.0.tgz
```

Helm Chart 公開のため Helm Chart リポジトリを新規に起動する場合、 `helm repo index [DIR]` により Helm リポジトリで公開する Chart のエントリを持つ Index ファイルを作成します。  
既存の Helm リポジトリの Index ファイルに新しく Helm Chart を追加する場合、 `--merge` オプションで Index ファイルを指定します。

```
$ helm repo index .
```

## カスタム Chart Repository の公開

静的コンテンツとして `index.yaml` および Chart のアーカイブを公開する Web サーバを起動すると Helm Reposytory として機能します。  
`httpd` Pod を起動し、 `index.yaml` および　Chart のアーカイブファイルをアップロードし `Route` 経由で公開します。

```
$ oc new-app --name custom-helm-repositry httpd
$ oc start-build httpd --from-dir . --wait
$ oc expose svc custom-helm-repositry
```

Helm リポジトリから Chart を取得しデプロイできることを確認するため、あらかじめ既存の Helm リリースを削除しておきます。

```
$ helm uninstall handson-$HANDSONUSER
```

Helm CLI 上で Helm リポジトリを追加し、 Chart の情報を確認できること、アプリケーションをデプロイできることを確認します。

```
$ helm repo add myrepo <Repo_URL>
$ helm search repo handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER myrepo/handson-$HANDSONUSER
```

## カスタム Chart Repository の Developer Catalog への追加

作成した Helm Chart を OpenShift の Web Console (Developer Perspective) から利用できるように、カスタム Helm リポジトリを Developer Catalog へ追加します。

以下のいずれかの方法でカスタム Helm リポジトリを Developer Catalog へ追加します。今回は 2 の方法で追加します。

1. クラスタ管理者が OpenShift クラスタ全体に Helm リポジトリを公開する (クラスタスコープの `HelmChartRepository` CR を作成する)
2. 適切な RBAC 権限を持つ Project メンバが特定の Namespace に限定して Helm リポジトリを公開する (Namespace スコープの ProjectHelmChartRepository CR を作成する)

`ProjectHelmChartRepository` CR の `url` に追加する Helm リポジトリの URL を指定します。

```yaml
# ProjectHelmChartRepository CR の例
apiVersion: helm.openshift.io/v1beta1
kind: ProjectHelmChartRepository
metadata:
  name: <name>
spec:
  url: https://my.chart-repo.org/stable

  # optional name that might be used by console
  name: <chart-repo-display-name>

  # optional and only needed for UI purposes
  description: <My private chart repo>

  # required: chart repository URL
  connectionConfig:
    url: <helm-chart-repository-url>
```

`prj-helm-chart-repo.yaml` の `url` を先ほど起動した Helm リポジトリの URL に変更し、 CR を作成します。

```yaml
apiVersion: helm.openshift.io/v1beta1
kind: ProjectHelmChartRepository
metadata:
  name: myrepo
spec:
  name: myrepo
  connectionConfig:
    url: <your-helm-chart-repository-url> # 変更
```

```
$ oc apply -f prj-helm-chart-repo.yaml 
```

OpenShift Web Console の Developer Perspective 上で [+Add] > [Helm Chart] を表示し、 `Myrepo` リポジトリが利用できることを確認します。

Web Console 上からデプロイできることを確認するため、あらかじめ既存の Helm リリースを削除しておきます。

```
$ helm uninstall handson-$HANDSONUSER
```

以下の手順で Web Console から Helm Chart をデプロイできることを確認します。

1. [Chart Repositories] で `Myrepo` にチェック
2. [Handson <ハンズオンユーザ名>] Chart を選択、 [Install Helm Chart] > [Install]

## Helm Dependency

アプリケーションの起動に必要なデータベースや依存サービスなど依存関係にある Helm Chart を同時にデプロイしたい場合、 `Chart.yaml` の `dependencies` フィールドに依存関係を定義します。詳細は [公式ドキュメント](https://helm.sh/docs/chart_best_practices/dependencies/) で確認できます。

```yaml
dependencies:
- name: nodejs-ex-k
  version: 0.2.1 # 最初に ~ をつけることで特定バージョン以上の Chart を利用するよう指定可能
  repository: https://redhat-developer.github.io/redhat-helm-charts
  condition: nodejs-ex-k.enabled # 該当するパラメータが false の場合依存 Chart をインストールしない
```

`Chart.yaml` に上の定義を追加し、 実際に同時にデプロイされることを確認します。  
事前に依存関係にある Helm Chart を `charts/` 配下にダウンロードするため `helm dependency update` を実行します。

```
$ helm lint ./handson-$HANDSONUSER
$ helm dependency update handson-$HANDSONUSER
# 実際の適用前のレンダリング時点のマニフェスト確認
$ helm upgrade --install --dry-run handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deploymentconfig nodejs-example
```

依存関係にある Chart のパラメータは `values.yaml` で `<Chart 名>.<パラメータフィールド>` という形で定義できます。

```yaml
nodejs-ex-k:
  enabled: true
  ingress:
    enabled: true
```

```
$ helm lint ./handson-$HANDSONUSER
# 実際の適用前のレンダリング時点のマニフェスト確認
$ helm upgrade --install --dry-run handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ oc get deploymentconfig nodejs-example
```

以上で Helm Chart 開発ハンズオンは終了です。お疲れ様でした。
