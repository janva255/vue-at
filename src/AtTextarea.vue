<script>
import At from './At.vue'
import getCaretCoordinates from 'textarea-caret'
import { getAtAndIndex } from './util'

export default {
  extends: At,
  name: 'AtTextarea',

  computed: {
    style () {
      if (this.atwho) {
        const { list, cur, x, y } = this.atwho
        const { wrap } = this.$refs
        const el = this.$el.querySelector('textarea')
        if (wrap) {
          const left = x - el.scrollLeft + 'px'
          const top = y - el.scrollTop + 'px'
          return { left, top }
        }
      }
      return null
    }
  },

  methods: {
    handleDelete (e) {
      const el = this.$el.querySelector('textarea')
      const text = el.value.slice(0, el.selectionEnd)
      if (text) {
        const { atItems, members, deleteMatch, itemName } = this
        const { at, index } = getAtAndIndex(text, atItems)
        if (index > -1) {
          const chunk = text.slice(index + at.length, -1)
          const has = members.some(v => {
            const name = itemName(v)
            return deleteMatch(name, chunk)
          })
          if (has) {
            el.value = el.value.slice(0, index) + el.value.slice(el.selectionEnd - 1)
            el.selectionStart = index + 1
            el.selectionEnd = index + 1
          }
        }
      }
    },

    handleInput () {
      if (this.hasComposition) return
      const el = this.$el.querySelector('textarea')
      const text = el.value.slice(0, el.selectionEnd)
      if (text) {
        const { atItems, avoidEmail, allowSpaces } = this
        let show = true
        const { at, index } = getAtAndIndex(text, atItems)
        if (index < 0) show = false
        const prev = text[index - 1]
        const chunk = text.slice(index + at.length, text.length)
        if (avoidEmail) {
          // 上一个字符不能为字母数字 避免与邮箱冲突
          // 微信则是避免 所有字母数字及半角符号
          if (/^[a-z0-9]$/i.test(prev)) show = false
        }
        if (!allowSpaces && /\s/.test(chunk)) {
          show = false
        }
      
        // chunk以空白字符开头不匹配 避免`@ `也匹配
        if (/^\s/.test(chunk)) show = false
        if (!show) {
          this.closePanel()
        } else {
          const { members, filterMatch, itemName } = this
          const matched = members.filter(v => {
            const name = itemName(v)
            return filterMatch(name, chunk, at)
          })
          if (matched.length) {
            this.openPanel(matched, chunk, index, at)
          } else {
            this.closePanel()
          }
        }
      }
    },

    openPanel (list, chunk, offset, at) {
      const fn = () => {
        const el = this.$el.querySelector('textarea')
        const end = offset + at.length // 从@后第一位开始
        const rect = getCaretCoordinates(el, end)
        this.atwho = {
          chunk,
          offset,
          list,
          x: rect.left,
          y: rect.top - 4,
          cur: 0, // todo: 尽可能记录
        }
      }
      if (this.atwho) {
        fn()
      } else { // 焦点超出了显示区域 需要提供延时以移动指针 再计算位置
        setTimeout(fn, 10)
      }
    },

    insertItem () {
      const { chunk, offset, list, cur } = this.atwho
      const { atItems, itemName } = this
      const el = this.$el.querySelector('textarea')
      const text = el.value.slice(0, el.selectionEnd)
      const { at, index } = getAtAndIndex(text, atItems)
      const start = index + at.length // 从@后第一位开始
      el.value = el.value.slice(0, start) + el.value.slice(start + chunk.length)
      el.selectionStart = start
      el.selectionEnd = start
      el.focus()
      document.execCommand('insertText', 0, itemName(list[cur]) + ' ')
    }
  }
}
</script>
