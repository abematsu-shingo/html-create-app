<script setup lang="ts">
import { nextTick, onMounted, ref } from 'vue'

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

// 選択範囲がeditArea内にあるか確認する関数
const isEditArea = (selection: Selection | null): boolean => {
  if (!selection || !editValue.value || selection.rangeCount === 0) return false
  const range = selection.getRangeAt(0)
  return editValue.value.contains(range.commonAncestorContainer)
}

// ユーザーの入力内容をプレビューに反映
const onInput = (e: Event) => {
  // const targetElement = e.target as HTMLElement
  // previewValue.value = targetElement.innerHTML

  if (!editValue.value) return // editValueがnullの場合は何もしない

  // カーソルの位置を保存
  const selection = getSelection()
  let savedRange: Range | null = null
  if (selection && selection.rangeCount > 0) {
    savedRange = selection.getRangeAt(0).cloneRange()
  }

  // DOMが更新される前にdivをpに変換
  normalizeDivToP()

  // nextTickでDOM更新後にプレビューとカーソル位置を更新
  nextTick(() => {
    if (editValue.value) {
      // プレビューの更新
      previewValue.value = editValue.value.innerHTML
    }
    // カーソル位置の復元
    if (savedRange && selection) {
      // カーソル位置と選択範囲がある場合
      try {
        selection.removeAllRanges() // 既存の選択範囲をクリア
        selection.addRange(savedRange) // 保存した範囲を再設定
      } catch (error) {
        // 何らかの理由でカーソル位置の復元に失敗した場合
        console.error('カーソル位置の復元に失敗:', error)

        // カーソルをeditValueの末尾に移動
        const newrange = document.createRange()
        if (editValue.value) {
          newrange.selectNodeContents(editValue.value)
          newrange.collapse(false) // カーソルを末尾に移動
          selection.removeAllRanges()
          selection.addRange(newrange)
        }
      }
    }
  })
}

// 現在の選択範囲を取得する関数
const getSelection = () => {
  if (window.getSelection) {
    return window.getSelection()
  }
  return null
}

// editValue内のdivタグをpタグに変換する関数
const normalizeDivToP = () => {
  if (!editValue.value) return // editValueがnullの場合は何もしない
  // 取得時以降のDOM操作の影響を受けないよう、StaticNodeList(静的な配列)を使用
  const divs = Array.from(editValue.value.querySelectorAll('div'))
  divs.forEach((div) => {
    // Enterキーで自動生成されるeditValue直下のdivタグをpタグに変換
    if (div.parentNode === editValue.value) {
      const p = document.createElement('p')
      // divの子要素をpに移動
      while (div.firstChild) {
        p.appendChild(div.firstChild)
      }
      // divをpに置き換える
      div.parentNode?.replaceChild(p, div)
    }
  })
}

/**
 * スタイルタグを、解除/適用する処理
 */
// styleType クリックされたスタイル適用ボタン
const onStyle = (styleType: 'bold' | 'italic' | 'underline' | 'strikeThrough') => {
  // ユーザーの選択範囲
  const selection = getSelection()
  if (!isEditArea(selection) || !selection) return // editArea外なら何もしない

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
  if (!isEditArea(selection) || !selection) return // editArea外なら何もしない

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
      break
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
  if (!isEditArea(selection) || !selection) return // editArea外なら何もしない

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

  // 既存のブロック要素が見つかり、それがクリックされたheadingTypeと同じであれば解除（Pタグに変換）
  if (
    blockElement &&
    blockElement !== editValue.value &&
    blockElement.nodeName.toLowerCase() === headingType
  ) {
    const newParagraph = document.createElement('p')
    // ブロック要素の内容を新しいPタグに移動
    while (blockElement.firstChild) {
      newParagraph.appendChild(blockElement.firstChild)
    }
    blockElement.parentNode?.replaceChild(newParagraph, blockElement)

    // カーソルを新しいPタグの中に維持
    const newRange = document.createRange()
    newRange.selectNodeContents(newParagraph)
    newRange.collapse(false) // カーソルをブロックの末尾に移動
    selection.removeAllRanges()
    selection.addRange(newRange)
  } else if (
    !blockElement ||
    blockElement === editValue.value ||
    !['P', 'H1', 'H2', 'H3'].includes(blockElement.nodeName)
  ) {
    // editValueの直下、P, H1, H2, H3 以外のブロック要素（LIなど）の場合、新しい headingType のタグを作成して適用
    const newBlock = document.createElement(headingType)
    if (blockElement && blockElement !== editValue.value) {
      // 既存のブロック要素の内容を新しい見出しタグに移動
      while (blockElement.firstChild) {
        newBlock.appendChild(blockElement.firstChild)
      }
      blockElement.parentNode?.replaceChild(newBlock, blockElement)
    } else {
      // 要素自体が存在しない場合、空のブロック要素を作成
      newBlock.appendChild(range.extractContents())
      range.insertNode(newBlock)
    }

    // カーソルを新しいブロック要素の中に維持
    const newRange = document.createRange()
    newRange.selectNodeContents(newBlock)
    newRange.collapse(false) // カーソルをブロックの末尾に移動
    selection.removeAllRanges()
    selection.addRange(newRange)
  } else {
    // 既存のブロック要素がP, H1, H2, H3 のいずれかで、クリックされた headingType と異なる場合、タグを置き換える
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

// リストのスタイルを適用/解除する
const onListStyle = (listType: 'unordered' | 'ordered') => {
  const selection = getSelection()
  if (!isEditArea(selection) || !selection) return // editArea外なら何もしない

  const range = selection.getRangeAt(0)
  const selectedText = range.toString()

  let commonAncestor = range.commonAncestorContainer
  if (commonAncestor.nodeType === Node.TEXT_NODE) {
    commonAncestor = commonAncestor.parentNode!
  }

  if (!(commonAncestor instanceof HTMLElement)) return

  // リスト要素を見つける
  let listElement: HTMLUListElement | HTMLOListElement | null = null
  let listItemElement: HTMLElement | null = null

  // LI要素、またはUL/OL要素から親を探す
  let currentElement: HTMLElement | null = commonAncestor
  while (currentElement && currentElement !== editValue.value) {
    if (currentElement.nodeName === 'LI') {
      listItemElement = currentElement
    } else if (currentElement.nodeName === 'UL' || currentElement.nodeName === 'OL') {
      listElement = currentElement as HTMLUListElement | HTMLOListElement
      break
    }
    currentElement = currentElement.parentElement
  }

  if (listElement && listItemElement) {
    // 既存のリスト要素が見つかった場合
    if (
      (listType === 'unordered' && listElement.nodeName === 'UL') ||
      (listType === 'ordered' && listElement.nodeName === 'OL')
    ) {
      // 同じタイプのリストが選択されている場合、解除
      const parent = listElement.parentNode
      if (parent) {
        // LIの内容をPタグに変更してリストを解除
        const newParagraph = document.createElement('p')
        while (listItemElement.firstChild) {
          newParagraph.appendChild(listItemElement.firstChild)
        }
        // UL/OLタグの前に挿入
        parent.insertBefore(newParagraph, listElement)
        // リスト要素を削除
        listItemElement.remove()
        // リスト要素が空になった場合、リスト自体を削除
        if (listElement.children.length === 0) {
          listElement.remove()
        }

        // カーソルを新しいPタグの末尾に移動
        const newRange = document.createRange()
        newRange.selectNodeContents(newParagraph)
        newRange.collapse(false) // カーソルをブロックの末尾に移動
        selection.removeAllRanges()
        selection.addRange(newRange)
      }
    } else {
      // 異なるタイプのリストが選択されている場合、リストを変更
      const newListTag = listType === 'unordered' ? 'UL' : 'OL'
      const newList = document.createElement(newListTag)

      // 現在のli要素の内容を新しいリストに移動
      const currentListItem = document.createDocumentFragment()
      while (listItemElement.firstChild) {
        currentListItem.appendChild(listItemElement.firstChild)
      }
      // 新しいリストアイテムを作成
      const newListItem = document.createElement('li')
      newListItem.appendChild(currentListItem)
      newList.appendChild(newListItem)

      // 既存のLI要素を新しいリストに移動
      listElement.parentNode?.replaceChild(newList, listElement)
      // カーソルを新しいli要素の末尾に移動
      const newRange = document.createRange()
      newRange.selectNodeContents(newListItem)
      newRange.collapse(false) // カーソルをブロックの末尾に移動
      selection.removeAllRanges()
      selection.addRange(newRange)
    }
  } else {
    // リスト要素が見つからない場合、新しいリストを作成
    const newListTag = listType === 'unordered' ? 'UL' : 'OL'
    const newList = document.createElement(newListTag)
    const newListItem = document.createElement('li')

    // 要素にカーソルがある場合、リストを作成
    let newListElement: HTMLElement | null = commonAncestor
    while (newListElement.firstChild) {
      newListItem.appendChild(newListElement.firstChild)
    }
    newListElement.parentNode?.replaceChild(newListItem, newListElement)

    newList.appendChild(newListItem)
    range.insertNode(newList)

    // カーソルを新しいリストの末尾に移動
    const newRange = document.createRange()
    newRange.selectNodeContents(newListItem)
    newRange.collapse(false) // カーソルをブロックの末尾に移動
    selection.removeAllRanges()
    selection.addRange(newRange)
  }
  updateContent()
}

// 文字色を変更する
let colorValue = ref('#333333') // 初期値を設定
// カラー入力のイベントハンドラー
const onColorStyle = (event: Event) => {
  const target = event.target as HTMLInputElement
  colorValue.value = target.value
}
// 反映ボタンのクリックイベントハンドラー
const setColor = () => {
  const selection = getSelection()
  if (!isEditArea(selection) || !selection) return // editArea外なら何もしない

  const range = selection.getRangeAt(0)
  const selectedText = range.toString()

  if (selectedText.length === 0) {
    // 選択範囲がない場合は、カーソル位置に空の要素を挿入
    const span = document.createElement('span')
    span.setAttribute('style', `color: ${colorValue.value}`)
    span.appendChild(document.createTextNode('\uFEFF')) // ZWSを挿入
    range.insertNode(span)

    // カーソルを新しい要素の中に維持
    range.setStart(span.firstChild!, 1)
    range.collapse(true)
    selection.removeAllRanges()
    selection.addRange(range)
  } else {
    // 選択範囲がある場合は、spanタグで囲む
    const span = document.createElement('span')
    span.setAttribute('style', `color: ${colorValue.value}`)
    span.appendChild(range.extractContents()) // 選択範囲の内容をspanに挿入
    range.insertNode(span) // spanを選択範囲の位置に挿入
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
        <label>見出し：</label>
        <button @click="onHeaedingStyle('h1')">見出し1</button>
        <button @click="onHeaedingStyle('h2')">見出し2</button>
        <button @click="onHeaedingStyle('h3')">見出し3</button>

        <label>リスト：</label>
        <button @click="onListStyle('unordered')">通常リスト</button>
        <button @click="onListStyle('ordered')">番号リスト</button>

        <label>文字色：</label>
        <input type="color" id="color-picker" @input="onColorStyle($event)" value="#333333" />
        <button @click="setColor">反映</button>

        <label>その他：</label>
        <button @click="onStyle('bold')">太字</button>
        <button @click="onStyle('italic')">斜体</button>
        <button @click="onStyle('underline')">下線</button>
        <button @click="onStyle('strikeThrough')">取消し線</button>
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
