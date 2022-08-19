# Helm Chart Development Handson

## はじめに

このハンズオンは Helm Chart の開発方法を実際に体験して理解することを目的とします。

以下の流れで Helm Chart を作成していきます。

1. Chart の作成
2. パッケージング
3. Chart Repository の公開

## 事前準備

CodeReady Workspace にアクセスし、割り当てられたユーザ名でログインします。その後、 ターミナルを起動します。

- ワークスペース作成方法 : https://\<CodeReady Workspace URL>/f?url=https://github.com/k-srkw/accelerator-operator-helm-handson.git にアクセス
- ターミナル起動方法 : 「Terminal」タブ -> 「Open Terminal in specific container」 -> 「cli」

https://codeready-openshift-workspaces.apps.cluster-bzs75.bzs75.sandbox368.opentlc.com/f?url=https://github.com/k-srkw/accelerator-operator-helm-handson.git

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
$ cd charts
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
今回はシンプルに ConfigMap のみを作成する Chart を作成します。

```
$ cat << EOF > handson-$HANDSONUSER/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
data:
  myvalue: "Hello World"
EOF
```

作成できたら実際に OpenShift にデプロイできることを確認します。

```
$ oc new-project helm-handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm list
$ oc get cm mychart-configmap
```

この時点ではテンプレート化は行っておらず、単純に格納したマニフェストファイルがそのまま適用されます。

### マニフェストファイルのテンプレート化

#### 組み込みオブジェクト

`templates` 配下に格納したマニフェストファイルを適用できることを確認できたので、このマニフェストファイルをテンプレート化しリソース名や各フィールドをパラメータ化していきます。

例えば ConfigMap リソース名 `.metadata.name` を Helm リリース名を含む名前にしたい場合、 {{ .Release.Name }} テンプレートディレクティブを使います。

以下のように `{{  }}` で囲んだテンプレートディレクティブにマニフェストの一部を置き換えることで、 Helm がデプロイ時にテンプレートから実際にデプロイするマニフェストファイルを生成します。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
```

```
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm list
$ oc get cm handson-$HANDSONUSER-configmap
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
favoriteDrink: coffee
EOF
```

テンプレートを以下のように変更します。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favoriteDrink }}
```

values.yaml の値が反映されることを確認します。

```
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm list
$ oc get cm handson-$HANDSONUSER-configmap -oyaml
```

さらに values.yaml を構造化し、新しいキーを追加します。

```yaml
favorite:
  drink: coffee
  food: pizza
```

これに合わせてテンプレートも変更します。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink }}
  food: {{ .Values.favorite.food }}
```

values.yaml の値が反映されることを確認します。

```
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm list
$ oc get cm handson-$HANDSONUSER-configmap -oyaml
```

#### テンプレート関数、パイプラインの利用

Helm Chart では `.Values` オブジェクトなどから取得した値を加工したり、 k8s クラスタ内の情報を取得したりするために利用できるテンプレート関数が用意されています。
例えば `quote` 関数を利用することで値を引用符で囲むことができます。

また、これらの関数はパイプラインで連結できるようになっており、以下は同じ定義となります。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ quote .Values.favorite.drink }}
  food: {{ quote .Values.favorite.food }}
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | quote }}
```

パイプラインの例として `upper` 関数を追加し連携させます。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
```

テンプレート関数により値が加工されることを確認します。

```
$ helm upgrade --install handson-$HANDSONUSER ./handson-$HANDSONUSER
$ helm list
$ oc get cm handson-$HANDSONUSER-configmap -oyaml
```

そのほかにも values.yaml に定義がない場合のデフォルト値を設定する `default` 関数や、 k8s クラスタ上のリソースを参照する `lookup` 関数が利用できます。

Helm には 60 以上の関数が用意されています。Helm は [Go テンプレート言語](https://pkg.go.dev/text/template) がベースとなっており、
そのうちのいくつかは、 Go テンプレート言語自体で定義されています。その他のほとんどは [Sprig](https://masterminds.github.io/sprig/) テンプレートライブラリの一部です。

[公式ドキュメント](https://helm.sh/docs/chart_template_guide/function_list/) で利用可能な関数を確認できます。

#### フロー制御

Helm テンプレートでは If / Else による条件分岐が利用できます。
ここで `eq` は値が同一かどうかを比較する関数です。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{ if eq .Values.favorite.drink "coffee" }}
  mug: "true"
  {{ end }}
```

上記をそのまま実行するとレンダリング時に `if` の部分が空行となります。
`{{- ` と中括弧に続けてハイフンを追加することで空行を詰めることができます。
下側を詰めたい場合は ` -}}` とします。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | default "tea" | quote }}
  food: {{ .Values.favorite.food | upper | quote }}
  {{- if eq .Values.favorite.drink "coffee" }}
  mug: "true"
  {{- end }}
```

オブジェクトのスコープの変更には `with` を利用できます。

例えば以下の例の場合、 `with` アクションの範囲内では各フィールドで毎回 `.Values.favorite` と書くことなくオブジェクトを利用できます。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
```

ただし注意点として親オブジェクトより上のスコープのオブジェクトにはアクセスできなくなります。
例えば以下の `.Release.Name` にはアクセスできずレンダリングに失敗します。

```yaml
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  release: {{ .Release.Name }}
```

これを解決するには　`.Release.Name` の前に `$` をつけるか、変数を利用します。

`$` から始まるオブジェクトはルートスコープからの参照とみなされ、 `with` 内であっても参照が可能になります。

また、以下のように `with` のスコープの外で変数を定義することでも解決できます。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- $relname := .Release.Name -}}
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  release: {{ $relname }}
  {{- end }}
```

#### 名前付きテンプレート (_helpers.tpl)

プレフィクスに `_` がつくファイルはマニフェストを持たないテンプレートファイルとみなされ、各テンプレートの共通処理の定義に利用されます。

以下のように `define` 関数でテンプレート間でグローバルによび出し可能なテンプレートを定義できます。

```yaml
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}"
{{- end -}}
```

呼び出すときには `include` 関数を利用します。 `indent` 関数で挿入されるフィールドのインデントを変更できます。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
data:
  myvalue: "Hello World"
  {{- range $key, $val := .Values.favorite }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
{{ include "mychart.app" . | indent 2 }}
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

## パッケージング

`helm package <CHART-PATH>` で Chart をパッケージングします。次に、Index ファイルを作成します。

```
$ helm package handson-$HANDSONUSER
$ helm repo index .
```

## Chart Repository の公開

```
$ oc new-app --name httpdd httpd~./
$ oc expose svc httpdd
```

```
$ helm repo add myrepo <Repo_URL>
$ helm search repo handson-$HANDSONUSER
$ helm upgrade --install handson-$HANDSONUSER myrepo/handson-$HANDSONUSER
```
