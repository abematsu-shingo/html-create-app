<script setup lang="ts">
import { onMounted, ref } from 'vue'

// WYSIWYGエリアの中身
const editValue = ref<HTMLElement | null>(null)
// previewエリアの中身
const previewValue = ref('')

// マウント時にサンプル表示
onMounted(() => {
  if (editValue.value) {
    editValue.value.innerHTML = '<p>ここに文字を入力してください</p>'
    previewValue.value = editValue.value.innerHTML
    // console.log(previewValue.value)
  }
})

// 現在の選択範囲を取得する関数
const getSelection = () => {
  if (window.getSelection) {
    return window.getSelection()
  }
  return null
}

// テキストスタイルを適用
// styleType クリックされたスタイル適用ボタン
const onStyle = (styleType: 'bold' | 'italic' | 'underline' | 'strikeThrough') => {
  const selection = getSelection()
  // getSelection()がnull or カーソルが存在しない場合、何もしない
  if (!selection || selection.rangeCount === 0) return

  const range = selection.getRangeAt(0)
  const selectedText = range.toString()
  console.log(selectedText)
  console.log(range.commonAncestorContainer.parentElement)

  // // 選択範囲がない場合は、カーソルの位置に空の要素を挿入し、その中にカーソルを置く
  // if (selectedText.length === 0) {
  //   onEmptyStyle(styleType)
  //   return
  // }

  let tagName: string
  let cssProperty: string | null = null
  let cssValue: string | null = null

  switch (styleType) {
    // クリックされたボタンが「太字」だった場合
    // <strong></strong>
    case 'bold':
      tagName = 'strong'
      break
    // クリックされたボタンが「斜体」だった場合
    // <em></em>
    case 'italic':
      tagName = 'em'
      break
    // クリックされたボタンが「下線」だった場合
    // <span style="text-decoration: underline;"></span>
    case 'underline':
      tagName = 'span'
      cssProperty = 'textDecoration'
      cssValue = 'underline'
      break
    // クリックされたボタンが「取消し線」だった場合
    // <span style="text-decoration: line-through;"></span>
    case 'strikeThrough':
      tagName = 'span'
      cssProperty = 'textDecoration'
      cssValue = 'line-through'
      break
    // クリックされたボタンがどれにも当てはまらない場合、関数終了
    default:
      return
  }

  clickTag(range, tagName, cssProperty, cssValue)

  updateContent()
}

// クリックされたタグを適用
const clickTag = (
  range: Range,
  tagName: string,
  cssProperty: string | null = null,
  cssValue: string | null = null,
) => {
  // 指定されたtagNameでタグを作成
  const newNode = document.createElement(tagName)
  // console.log(newNode)
  if (cssProperty && cssValue) {
    // cssPropertyとcssValueが指定されていた場合、style指定
    ;(newNode.style as any)[cssProperty] = cssValue
  }
  // 選択範囲の要素を切り取り、newNodeの末尾に挿入
  newNode.appendChild(range.extractContents())
  // 上記で切り取られた後の選択範囲の最初にnewNodeを挿入
  range.insertNode(newNode)
}

// スタイルを適用した後、htmlプレビューを更新
const updateContent = () => {
  if (editValue.value) {
    previewValue.value = editValue.value.innerHTML
  }
}

const onInput = (e: Event) => {
  previewValue.value = (e.target as HTMLElement).innerHTML
}
</script>
<template>
  <h1>HTML Create App</h1>
  <div class="container">
    <div class="wysiwygArea">
      <h3>WYSIWYGエディタ</h3>
      <div class="toolbar">
        <label for="heading">見出し：</label>
        <button @click="onHeaedingStyle('h1')" id="heading">見出し1</button>
        <button @click="onHeaedingStyle('h2')" id="heading">見出し2</button>
        <button @click="onHeaedingStyle('h3')" id="heading">見出し3</button>

        <label for="list">リスト：</label>
        <button @click="onListStyle('unordered')" id="list">通常リスト</button>
        <button @click="onListStyle('ordered')" id="list">番号リスト</button>

        <label for="textColor">文字色：</label>
        <input type="color" @input="onColorStyle($event)" id="textColor" value="#333333" />

        <label for="other">その他：</label>
        <button @click="onStyle('bold')" id="other">太字</button>
        <button @click="onStyle('italic')" id="other">斜体</button>
        <button @click="onStyle('underline')" id="other">下線</button>
        <button @click="onStyle('strikeThrough')" id="other">取消し線</button>
      </div>
      <div contenteditable="true" @input="onInput" class="editArea" ref="editValue"></div>
    </div>

    <div class="previewArea">
      <h3>HTMLプレビュー</h3>
      <div class="htmlPreview">{{ previewValue }}</div>
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
