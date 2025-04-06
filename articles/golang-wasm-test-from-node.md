---
title: "【Golang】【WebAssembly】Goで生成したWasmをNode.jsでテストする"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["golang", "webassembly", "nodejs"]
published: true
published_at : "2025-04-06 21:30"
---
## WebAssemblyをNode.jsでテストする

Goが1.24になり、`go:wasmexport`でいい感じに関数を直接エクスポートできるようになりました[^1]。
そこで、WASMに固めたものをGo以外の言語でサクッとテストできないかなぁと考えていたんですが、GOOS=jsと指定しているので、Node.jsでテストするのが適切という結論になりました[^2]。

## サンプル作成

適当な値を出力するサンプルを作成します。

```go
package main

//go:wasmexport calculateSumWasm
func calculateSumWasm(iterations int32) int32 {
    var sum int32 = 0
    for i := int32(0); i < iterations; i++ {
        sum++
    }
    return sum
}

func main() {
}

```

これをjs向けにビルドします。

```bash
GOOS=js GOARCH=wasm go build -o wasm/main.wasm main.go
```

ブラウザ上で用いる場合には、`wasm_exec.js` が必要になります[^3]。
`$(go env GOROOT)/lib/
wasm/wasm_exec.js` から取得しましょう。

## テストひな形作成

Vitestで作成します。どうせならTypeScriptを使いたいので、Viteでひな形を作成してそのまま流用してしまいます。

```bash
$ npm create vite@latest wasm-vitest-test

> npx
> cva wasm-vitest-test

│
◇  Select a framework:
│  Others
│
◇  Select a variant:
│  Extra Vite Starters ↗

> npx
> create-vite-extra wasm-vitest-test

✔ Select a template: › library
✔ Select a variant: › TypeScript
```

<https://vitest.dev/guide/>にしたがって初期設定しましょう。

```bash
npm install -D vitest
```

`tests/wasm.test.ts`を作り、最低限のテストで動作確認します。

```bash
import { expect, test } from 'vitest'


test('initial', () => {
  expect(true).toBe(true);
})
```

## WASMに対してのテスト作成

適当なディレクトリに対して、ブラウザで使用するためのファイルを追加します。今回は`wasm`ディレクトリを作成します。

```bash
/wasm-vitest-test$ ls wasm/
main.wasm  wasm_exec.js
```

とりあえずjsで作成しています。
TypeScriptにする場合には、Goクラスの定義が必要なので`.d.ts`の追加が必要です。

```js
import { readFileSync } from 'fs';
import { expect, test } from 'vitest'
import * as wasmExec from '../wasm/wasm_exec.js'; // これは必須なので消さないこと。

const go = new Go();
const instancePromise = WebAssembly.instantiate(readFileSync('wasm/main.wasm'), go.importObject).then((result) => {
    const inst = result.instance;
    return go.run(inst).then(() => inst);
});

let instance;

test('calculateSumWasm', async () => {
    instance = await instancePromise;
    
    const iterations = 1000000;
    const sum = instance.exports.calculateSumWasm(iterations);

    expect(sum).toBe(1000000);
});
```

通常のテストと同様に`npm run test`で実行します。

```bash
 ✓ tests/wasm.test.js (1 test) 4ms
   ✓ calculateSumWasm 3ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  21:10:39
   Duration  271ms (transform 51ms, setup 0ms, collect 52ms, tests 4ms, environment 0ms, prepare 58ms)
```

## 参考

- [【Golang】【wasm】テストの仕方と注意点 @ Go 1.16+](https://qiita.com/KEINOS/items/0ab42c53dcebc5a925f0)

## 注釈

[^1]: tinygoは除きます。
[^2]: GOOS=wasip1を指定しないとダメなようですね。ビルドの方法が変わるのは本意ではないので除外します。
[^3]: <https://go.dev/wiki/WebAssembly#getting-started>
