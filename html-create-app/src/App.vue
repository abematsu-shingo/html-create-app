<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

// WYSIWY
const editArea = ref<HTMLElement | null>(null)
const foo = ref('')

onMounted(() => {
  if (editArea.value) {
    editArea.value.innerHTML = '<p>ここに文字を入力してください</p>'
    foo.value = editArea.value.innerHTML
    console.log(foo.value)
  }
})

const onInput = (e: Event) => {
  foo.value = (e.target as HTMLElement).innerHTML
}
</script>
<template>
  <h1>HTML Create App</h1>
  <div class="container">
    <div class="wysiwygArea">
      <h3>WYSIWYGエディタ</h3>
      <div class="toolbar">
        <label for="heading">見出し：</label>
        <button id="heading">見出し1</button>
        <button id="heading">見出し2</button>
        <button id="heading">見出し3</button>

        <label for="list">リスト：</label>
        <button id="list">通常リスト</button>
        <button id="list">番号リスト</button>

        <label for="textColor">文字色：</label>
        <input type="color" id="textColor" />

        <label for="other">その他：</label>
        <button id="other">太字</button>
        <button id="other">斜体</button>
        <button id="other">下線</button>
        <button id="other">取消し線</button>
      </div>
      <div contenteditable="true" @input="onInput" class="editArea" ref="editArea"></div>
    </div>

    <div class="previewArea">
      <h3>HTMLプレビュー</h3>
      <div class="htmlPreview">{{ foo }}</div>
    </div>
  </div>
</template>
<style scoped>
* {
  box-sizing: border-box;
}
h1 {
  text-align: center;
}
h3 {
  color: #333;
}

.container {
  display: flex;
  height: 85vh;
  gap: 10px;
}

.wysiwygArea,
.previewArea {
  flex: 1;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 10px;
  background-color: #f9f9f9;
  overflow: hidden;
  display: flex; /* 子要素のflex-glowを適用させるための設定 */
  flex-direction: column;
}

.editArea {
  min-height: 300px;
  border: 1px solid #ccc;
  margin-top: 10px;
  padding: 10px;
  background-color: #fff;
  flex-grow: 1; /* 残りのスペースを埋める */
  overflow-y: auto; /* はみ出したらスクロール */
  outline: none;
}
.toolbar {
  display: flex;
  flex-wrap: wrap; /* ボタンを改行 */
  gap: 5px;
  padding-bottom: 10px;
  border-bottom: 1px solid #ccc;
}
.toolbar button,
.toolbar input[type='color'] {
  border: 1px solid #ccc;
  border-radius: 8px;
  background-color: #bfe3e5;
  cursor: pointer;
  transition: background-color 0.2s;
}
.toolbar button:hover,
.toolbar input[type='color']:hover {
  background-color: #89dde0;
}
.toolbar input[type='color'] {
  padding: 2px 5px;
}
.toolbar label {
  font-size: 0.9em;
  align-self: center;
}

.htmlPreview {
  min-height: 300px;
  border: 1px solid #ccc;
  padding: 10px;
  background-color: #fff;
  flex-grow: 1;
  overflow-y: auto;
  white-space: pre-wrap; /* エディタの改行・スペースをそのまま表示 */
  word-break: break-all; /* 長い文章は折り返す */
}
</style>
