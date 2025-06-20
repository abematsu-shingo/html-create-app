<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

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

// ユーザーの入力内容をプレビューに反映
const onInput = (e: Event) => {
  previewValue.value = (e.target as HTMLElement).innerHTML
}
const foo = computed(() => {
  return previewValue.value
})

// 現在の選択範囲を取得する関数
const getSelection = () => {
  if (window.getSelection) {
    return window.getSelection()
  }
  return null
}

/**
 * スタイルタグを、解除/適用する処理
 */
// styleType クリックされたスタイル適用ボタン
const onStyle = (styleType: 'bold' | 'italic' | 'underline' | 'strikeThrough') => {
  // ユーザーの選択範囲
  const selection = getSelection()
  // getSelection()がnull or カーソルが存在しない場合、何もしない
  if (!selection || selection.rangeCount === 0) return

  // 選択範囲のオブジェクトデータ
  const range = selection.getRangeAt(0)
  // 選択範囲の文字列
  const selectedText = range.toString()
  // console.log(selectedText)
  // console.log(range.commonAncestorContainer.parentElement)

  // // 選択範囲がない場合は、カーソルの位置に空の要素を挿入し、その中にカーソルを置く
  if (selectedText.length === 0) {
    onEmptyStyle(styleType)
    return
  }

  // ボタンの引数から取得したパラメータ変数を変数に格納
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

  /**
   * 選択範囲の「共通の始祖要素」を確認し、DOM操作を行う要素の特定処理
   */
  // 共通の始祖要素を取得
  const commonAncestor = range.commonAncestorContainer
  // 共通の始祖要素全体を格納する変数
  let parentElement: HTMLElement | null = null
  // 選択範囲がテキストのみだった場合、テキストが属する要素を格納
  if (commonAncestor.nodeType === Node.TEXT_NODE) {
    parentElement = commonAncestor.parentElement
  }
  // 選択範囲にエレメントタグが含まれている場合、変数に親要素ごと格納
  else if (commonAncestor.nodeType === Node.ELEMENT_NODE) {
    parentElement = commonAncestor as HTMLElement
  }
  console.log(commonAncestor)
  console.log(parentElement)

  /**
   * 特定したDOM要素にすでにスタイルが適用されているかの確認し、スタイルの解除/適用を判別
   */
  // スタイルチェック関数を呼び出し、引数の文字列が含まれているか確認
  const isApplied = checkStyle(parentElement, tagName, cssProperty, cssValue)
  // スタイルの解除
  if (isApplied) {
    unSetStyle(range, tagName, cssProperty, cssValue)
  }
  // スタイルの適用
  else {
    setStyle(range, tagName, cssProperty, cssValue)
  }

  updateContent()
}

// スタイルが適用されているかの確認
const checkStyle = (
  parentElement: HTMLElement | null,
  tagName: string,
  cssProperty: string | null,
  cssValue: string | null,
): boolean => {
  if (!parentElement) return false

  /**
   * 祖先要素に指定のタグ・スタイルがあるかを確認
   * 祖先要素に指定タグ・スタイルが存在しない場合、子孫要素をに存在しているかを確認
   */
  // 祖先要素が指定のタグかつ、style適用済みかを確認
  if (parentElement.nodeName.toLowerCase() === tagName) {
    if (cssProperty && cssValue) {
      return (parentElement as HTMLElement).style[cssProperty as any] === cssValue
    }
    return true
  }

  // 祖先要素に指定のタグがない場合、子孫要素に指定のタグが存在するかを確認
  const elements = parentElement.querySelectorAll(tagName)
  for (const elem of Array.from(elements)) {
    if (rangeContainsNode(getSelection()!.getRangeAt(0), elem)) {
      if (cssProperty && cssValue) {
        if ((elem as HTMLElement).style[cssProperty as any] === cssValue) {
          return true
        }
      } else {
        return true
      }
    }
  }
  return false
}

// クリックされたタグを解除
const unSetStyle = (
  range: Range,
  tagName: string,
  cssProperty: string | null = null,
  cssValue: string | null = null,
) => {
  const commonAncestor = range.commonAncestorContainer
  let parentElement: HTMLElement | null = null
  if (commonAncestor.nodeType === Node.TEXT_NODE) {
    parentElement = commonAncestor.parentElement
  } else if (commonAncestor.nodeType === Node.ELEMENT_NODE) {
    parentElement = commonAncestor as HTMLElement
  }

  // palentElementに値が入っていなければ、関数の終了
  if (!parentElement) return

  /**
   * 指定タグの親要素を確認して、スタイルを解除する処理
   */
  // 取得した始祖要素（parentElement）をwhile文で再利用するために、変数を再定義
  let targetElemnt: HTMLElement | null = parentElement
  // targetElementの中身がユーザーの入力内容と一致しない場合
  while (targetElemnt && targetElemnt !== editValue.value) {
    // targetElementの中身のtag名とtagNameが一致する場合
    if (targetElemnt.nodeName.toLowerCase() === tagName) {
      // targetElementにstyleが設定されている場合
      if (cssProperty && cssValue) {
        // targetElementのstyleが引数と一致する場合
        if ((targetElemnt.style as any)[cssProperty] === cssValue) {
          // targetElementのタグが<span style="textDecoration: cssValue;">だった場合
          if (targetElemnt.nodeName.toLowerCase() === 'span' && cssProperty && cssValue) {
            // style削除
            ;(targetElemnt.style as any)[cssProperty] = ''
            // styleが削除されたら、タグを削除
            if (!targetElemnt.style.length) {
              unSetTag(targetElemnt)
            }
          } else {
            // <span>タグではない場合、タグを削除
            unSetTag(targetElemnt)
          }
          return
        }
        // styleが設定されていない場合、タグを削除
      } else {
        unSetTag(targetElemnt)
        return
      }
    }
    // タグを削除した中身をtargetElementに代入
    targetElemnt = targetElemnt.parentElement
  }
}

// 選択箇所のノードを解除し、親ノードの末尾に移動
const unSetTag = (node: Node) => {
  const parent = node.parentNode
  if (parent) {
    while (node.firstChild) {
      parent.insertBefore(node.firstChild, node)
    }
    parent.removeChild(node)
  }
}

// クリックされたタグを適用
const setStyle = (
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

/**
 * ノードが範囲内に含まれているかチェック
 */
const rangeContainsNode = (range: Range, node: Node) => {
  return range.comparePoint(node, 0) >= 0 && range.comparePoint(node, node.childNodes.length) <= 0
}

/**
 * ユーザーが範囲選択していない場合、カーソル位置に空の要素を挿入
 */
const onEmptyStyle = (styleType: 'bold' | 'italic' | 'underline' | 'strikeThrough') => {
  const selection = getSelection()
  if (!selection || selection.rangeCount === 0) return

  const range = selection.getRangeAt(0)
  let tagName: string
  let cssProperty: string | null = null
  let cssValue: string | null = null

  switch (styleType) {
    case 'bold':
      tagName = 'strong'
      break
    case 'italic':
      tagName = 'italic'
      break
    case 'underline':
      tagName = 'span'
      cssProperty = 'textDecoration'
      cssValue = 'underline'
    case 'strikeThrough':
      tagName = 'span'
      cssProperty = 'textDecoration'
      cssValue = 'line-through'
      break
    default:
      return
  }

  // ユーザーがクリックした該当ボタンの要素を生成
  const newNode = document.createElement(tagName)
  if (cssProperty && cssValue) {
    ;(newNode.style as any)[cssProperty] === cssValue
  }
  // 作成した要素にZWS(zero width space)を挿入してカーソルを保持
  newNode.appendChild(document.createTextNode('\uFEFF'))
  range.insertNode(newNode)

  /**
   * カーソルを新しいノードの中に挿入
   */
  // ZWSの後にカーソル
  range.setStart(newNode.firstChild!, 1)
  // range.collapse(true) 範囲選択の開始位置と終了位置が同じ
  range.collapse(true)
  // ブラウザによる範囲選択無視の機能対応のため、一度選択した範囲をリセット
  selection.removeAllRanges()
  // 選択範囲を再度指定
  selection.addRange(range)

  updateContent()
}

/**
 * 見出しのスタイルを適用する
 */
const onHeaedingStyle = (headingType: string) => {
  const selection = getSelection()
  if (!selection || selection.rangeCount === 0) return

  const range = selection.getRangeAt(0)
  let commonAncestor = range.commonAncestorContainer

  if (commonAncestor.nodeType === Node.TEXT_NODE) {
    commonAncestor = commonAncestor.parentNode!
  }

  if (!(commonAncestor instanceof HTMLElement)) return

  // 段落要素または見出し要素を見つける
  let blockElement: HTMLElement | null = commonAncestor
  while (
    blockElement &&
    blockElement !== editValue.value &&
    !['P', 'H1', 'H2', 'H3'].includes(blockElement.nodeName)
  ) {
    blockElement = blockElement.parentElement
  }

  if (!blockElement || blockElement === editValue.value) {
    // ブロック要素が見つからない場合、新しいPタグを作成して適用
    setStyle(range, headingType)
  } else {
    // 既存のブロック要素を新しいタグに置き換える
    const newBlock = document.createElement(headingType)
    while (blockElement.firstChild) {
      newBlock.appendChild(blockElement.firstChild)
    }
    blockElement.parentNode?.replaceChild(newBlock, blockElement)

    // カーソルを新しいブロック要素の中に維持
    const newRange = document.createRange()
    newRange.selectNodeContents(newBlock)
    newRange.collapse(false) // カーソルをブロックの末尾に移動
    selection.removeAllRanges()
    selection.addRange(newRange)
  }
  updateContent()
}

// スタイルを適用した後、htmlプレビューを更新
const updateContent = () => {
  if (editValue.value) {
    previewValue.value = editValue.value.innerHTML
  }
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
