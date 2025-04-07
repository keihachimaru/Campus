<template>
  <div id="editor">
    <div class="editor" contenteditable="true" ref="editor" spellcheck="false">
      <div
        v-for="(line, i) in lines" 
        :key="i"
        :data-index="i"
        :class="{'line':true,'focused':i==focused}">
        <div
          class="line-index"
          contenteditable="false">
          {{ i+1 }}
        </div>
        <div
          v-html="mdDisplay(line.content)"
          class="line-contents">
        </div>
      </div>
    </div>
  </div>
</template>

<script>

export default {
  name: 'kCampus',
  props: {
    text: String,
    params: Object,
    action: Object  
  },
  data() {
    return {
      content: '',
      lines: [
        {
          content: '',
        }
      ],
      focused: -1,
    }
  },
  methods: {
    getCursorPosition(parent, node, offset, stat) {
      if (stat.done) return stat;

      let currentNode = null;
      if (parent.childNodes.length == 0) {
          stat.pos += parent.textContent.length;
      } else {
          for (let i = 0; i < parent.childNodes.length && !stat.done; i++) {
          currentNode = parent.childNodes[i];
          if (currentNode === node) {
              stat.pos += offset;
              stat.done = true;
              return stat;
          } else this.getCursorPosition(currentNode, node, offset, stat);
          }
      }
      return stat;
    },
    setCursorPosition(parent, range, stat) {
      if (stat.done) return range;

      if (parent.childNodes.length == 0) {
          if (parent.textContent.length >= stat.pos) {
          range.setStart(parent, stat.pos);
          stat.done = true;
          } else {
          stat.pos = stat.pos - parent.textContent.length;
          }
      } else {
          for (let i = 0; i < parent.childNodes.length && !stat.done; i++) {
              let currentNode = parent.childNodes[i];
              this.setCursorPosition(currentNode, range, stat);
          }
      }
      return range;
    },
    getCaretGlobalPosition() {
      const selection = document.getSelection();
      
      if (!selection.rangeCount) {
          return null;
      }
      
      const range = selection.getRangeAt(0);
      const node = range.startContainer;
      const offset = range.startOffset;
      let rect, r2;

      if (offset > 0) {
          r2 = document.createRange();
          r2.setStart(node, offset - 1);
          r2.setEnd(node, offset);
          rect = r2.getBoundingClientRect();
          return { left: rect.right , top: rect.bottom};
      }
      else {
          rect = node.getBoundingClientRect();
          return { left: rect.left, top: rect.top + 30, begginning: true};
      }
    },
    mdDisplay(content) {
      if(!content) return ''
      let heading = /^(#+)\s+(.*)/
      if(content.match(heading)) {
        let match = content.match(heading)
        let num = match[1].length>6?6:match[1].length
        return `<span class="md-header-${num} md-code">${match[1]}&nbsp;</span><span class="md-header-${num}">${match[2]}</span>`
      }
      return content
    },
    findSelectedDivs(node, range) {
        const selectedDivs = [];
        const walker = document.createTreeWalker(node, NodeFilter.SHOW_ELEMENT, null, false);

        let currentNode = walker.nextNode();

        while (currentNode) {
            if (currentNode.nodeName === 'DIV') {
                const nodeRange = document.createRange();
                nodeRange.selectNodeContents(currentNode);

                if (range.intersectsNode(currentNode)) {
                    selectedDivs.push(currentNode);
                }

                if (nodeRange.compareBoundaryPoints(Range.END_TO_END, range) >= 0) {
                    break;
                }
            }

            currentNode = walker.nextNode();
        }
        return selectedDivs
    },
    getCurrentWord() {
        const selection = window.getSelection();
        const range = selection.getRangeAt(0);
        
        const startContainer = range.startContainer;
        const startOffset = range.startOffset;
        
        if (startContainer.nodeType === Node.TEXT_NODE) {
            const text = startContainer.textContent;
            let wordStart = startOffset;
            let wordEnd = startOffset;
            
            while (wordStart > 0 && text[wordStart - 1].match(/\w/)) {
            wordStart--;
            }
            
            while (wordEnd < text.length && text[wordEnd].match(/\w/)) {
            wordEnd++;
            }
            
            return text.substring(wordStart, wordEnd);
        }
        
        return "";
    },
    getLineDisplay(contents) {
      let heading = /^(#+)\s/
      if(contents.match(heading)) {
        let match = contents.match(heading)
        let tag = 'h'+(match[1].length>6?6:match[1].length).toString()
        return `<${tag}>${contents}</${tag}>`
      }
      let tag = 'span'
      return `<${tag}>${contents}</${tag}>`
    },
    centerEditor() {
      let element = this.getLineByIndex(this.focused)
      const container = this.$refs.editor
      const containerHeight = container.clientHeight
      const elementTop = element.offsetTop
      const elementHeight = element.clientHeight

      const scrollPosition = elementTop - (containerHeight / 2) + (elementHeight / 2);

      container.scrollTop = scrollPosition;
    },
    updateContent(e) {
      this.content = this.lines.map((l) => {
          let normalizedStr = l.content.replace(/^[\s\u00A0\u2000-\u200F\u202F\u205F\u3000]+/g, match => ' '.repeat(match.length))
          let leadingSpaces = normalizedStr.match(/^ */)[0].length / 4
          return '\t'.repeat(leadingSpaces) + l.content.trim()
      }).join('\n')

      if(!e) {
        this.$emit('update', this.content.replace(/\u00A0/g, ' '))
        return
      }
      e.preventDefault()
      this.$emit('update', this.content)
      let lastLine = this.$refs.textarea.value.split('\n').at(-1)
      let tabCount = lastLine.length - lastLine.replace('\t','').length
      tabCount = lastLine.trim()!=''?tabCount:0
      var start = this.$refs.textarea.selectionStart;
      var end = this.$refs.textarea.selectionEnd;

      // set textarea value to: text before caret + tab + text after caret
      this.$refs.textarea.value = this.$refs.textarea.value.substring(0, start) +
        "\n"+"\t".repeat(tabCount) + this.$refs.textarea.value.substring(end);

      // put caret at right position again
      this.$refs.textarea.selectionStart =
      this.$refs.textarea.selectionEnd = start + 1 + tabCount;
    },
    getLineByIndex(i) {
      let lines = Array.from(this.$refs.editor.getElementsByClassName('line'))
      return lines[i]
    },
    focusLine(index, pos) {
      let focusRange
      if(window.getSelection().focusNode) {
        focusRange = window.getSelection().getRangeAt(0)
      }
      else {
        focusRange = document.createRange()
      }
      let line = this.getLineByIndex(index).querySelector('.line-contents')
      const sel = window.getSelection()
      const range = this.setCursorPosition(line, focusRange, {
        pos: pos!=undefined?pos:this.lines[index].content.length,
        done: false,
      });
      range.collapse(true);
      sel.addRange(range);
    }
  },
  mounted() {
    this.content = this.text
    this.lines = this.text.split('\n').map(l=>{return {'content': 'Hola'+l.replace('\t','\u00A0\u00A0\u00A0\u00A0')}})
    const selection = window.getSelection();
    selection.removeAllRanges();
    this.focused = 0

    this.$nextTick(()=>{
      this.focusLine(this.focused)
      this.$refs.editor.style.scrollBehavior = 'smooth'
      this.centerEditor()
      this.$refs.editor.style.scrollBehavior = 'auto'
    })
    
    this.$refs.editor.addEventListener('mousedown', (e)=>{
      let target = e.target
      let indexPart = target.classList.contains('line-index')
      while(!target.classList.contains('line')&&target!=this.$refs.editor) {
        target = target.parentElement
      }
      if(target.classList.contains('line')) {
        this.focused = parseInt(target.getAttribute('data-index'))
        if(indexPart) {
          e.preventDefault()
          this.focusLine(this.focused)
        }
        this.$refs.editor.style.scrollBehavior = 'smooth'
        this.centerEditor()
        this.$refs.editor.style.scrollBehavior = 'auto'
      }
      else {
        if(this.focused>=0) {
          e.preventDefault()
          this.focused = -1
          const selection = window.getSelection();
          selection.removeAllRanges();
        }
        else {
          e.preventDefault()
          this.focused = this.lines.length-1
          this.focusLine(this.focused)
        }
      }
    })
    this.$refs.editor.addEventListener('keydown', (e)=>{
      try {
        let selectedLines = this.findSelectedDivs(this.$refs.editor, window.getSelection().getRangeAt(0)).filter(e => e.classList.contains('line'))
        if(selectedLines.length==1) {
          if(e.key=='Enter') {
            e.preventDefault()
            this.focused++
            let s = 0
            while(this.lines[this.focused-1].content[s]&&this.lines[this.focused-1].content[s].trim()=='') {
              s ++;
            }
            console.log(s)
            if(this.lines[this.focused-1].content.trim()=='') s = 0
            this.lines.splice(this.focused, 0, {
              content: '\u00A0'.repeat(s)
            })
            this.$nextTick(()=>{
              this.focusLine(this.focused)
              this.centerEditor()
            })
          }
          else if(e.key=='Backspace') {
            if(this.lines[this.focused].content.length==0) {
              if(this.lines.length==1) {
                e.preventDefault()
              }
              else {
                e.preventDefault()
                this.lines.splice(this.focused, 1)
                this.focused--
                this.$nextTick(()=>{
                  this.focusLine(this.focused)
                })
              }
            }
          }
          else if(e.key=='Tab') {
            e.preventDefault()
            let selection = window.getSelection();
            let range = selection.getRangeAt(0);

            let start = range.startOffset;
            let end = range.endOffset;

            if(start==end) {
              this.lines[this.focused].content = this.lines[this.focused].content.substring(0, start) +
              "\u00A0\u00A0\u00A0\u00A0" + this.lines[this.focused].content.substring(end);
            }
            else {
              this.lines[this.focused].content = "\u00A0\u00A0\u00A0\u00A0"+this.lines[this.focused].content
            }
            this.$nextTick(()=>{
              this.focusLine(this.focused, end+4)
            })
          }
          else if(e.key.startsWith('Arrow')) {
            if(e.key=='ArrowUp') {
              this.focused = this.focused>0?this.focused-1:0
            }
            else if(e.key=='ArrowDown') {
              this.focused = this.focused==this.lines.length-1?this.focused:this.focused+1
            }
            else if(e.key=='ArrowLeft') {
              let selectedLine = this.findSelectedDivs(this.$refs.editor, window.getSelection().getRangeAt(0)).filter(e => e.classList.contains('line'))[0]
              let lineContent = selectedLine.querySelector('.line-contents')

              const sel = window.getSelection();
              const node = sel.focusNode;
              const offset = sel.focusOffset;
              const pos = this.getCursorPosition(lineContent, node, offset, { pos: 0, done: false })
              
              if(!selectedLine.classList.contains('focused') || 
                (pos.pos==0 && this.focused>0)) {
                this.focused --
              }
            }
            else if(e.key=='ArrowRight') {
              let selectedLine = this.findSelectedDivs(this.$refs.editor, window.getSelection().getRangeAt(0)).filter(e => e.classList.contains('line'))[0]
              let lineContent = selectedLine.querySelector('.line-contents')
              
              const sel = window.getSelection();
              const node = sel.focusNode;
              const offset = sel.focusOffset;
              const pos = this.getCursorPosition(lineContent, node, offset, { pos: 0, done: false })

              if(!selectedLine.classList.contains('focused') || 
                (this.lines[this.focused].content.length==pos.pos && this.focused<(this.lines.length-1))) {
                this.focused ++
              }
            }
            this.centerEditor()
          }
        }
        else if (selectedLines.length>1){
          const sel = window.getSelection();
          const range = sel.getRangeAt(0);
          const firstLine = range.startContainer;
          const lastLine = range.endContainer;

          const firstLineIndex = range.startOffset + selectedLines.at(0).textContent.length - firstLine.length - 1;
          const lastLineIndex = range.endOffset + selectedLines.at(-1).textContent.length - lastLine.length - 1;

          if(e.key=='Enter') {
            e.preventDefault()
            let delStart = parseInt(selectedLines[0].getAttribute('data-index'))
            let delCount = selectedLines.length
            let newFocused = delStart
            let lastLineNum = delStart+delCount-1
            
            if(selectedLines.length>2) {
              let removed = this.lines.splice(delStart+1,delCount-2)
              lastLineNum -= removed.length
            }

            this.lines[newFocused].content = this.lines[newFocused].content.slice(0, firstLineIndex)
            this.lines[lastLineNum].content = this.lines[lastLineNum].content.slice(lastLineIndex)
            this.focused = lastLineNum
            sel.removeAllRanges()
            this.focusLine(this.focused)
          }
          else if(e.key=='Backspace') {
            e.preventDefault()
            let delStart = parseInt(selectedLines[0].getAttribute('data-index'))
            let delCount = selectedLines.length
            let newFocused = delStart
            let lastLineNum = delStart+delCount-1

            this.lines[newFocused].content = this.lines[newFocused].content.slice(0, firstLineIndex)+this.lines[lastLineNum].content.slice(lastLineIndex)
            
            let removed = this.lines.splice(delStart+1,delCount-1)
            lastLineNum -= removed.length

            this.focused = lastLineNum
            sel.removeAllRanges()
            this.$nextTick(()=>{
              this.focusLine(this.focused, firstLineIndex)
            })
          }
          else if(e.key=='Tab') {
            e.preventDefault()
            this.focused = parseInt(selectedLines.at(0).getAttribute('data-index'))

            for(let i=0;i<selectedLines.length;i++) {
              this.lines[this.focused].content = "\u00A0\u00A0\u00A0\u00A0"+this.lines[this.focused].content
              this.focused ++
            }

            this.focused -= selectedLines.length
            this.$nextTick(()=>{
              this.focusLine(this.focused, firstLineIndex+4)
            })
          }
          else if(e.key.length==1) {
            e.preventDefault()
            let delStart = parseInt(selectedLines[0].getAttribute('data-index'))
            let delCount = selectedLines.length
            let newFocused = delStart
            let lastLineNum = delStart+delCount-1

            this.lines[newFocused].content = this.lines[newFocused].content.slice(0, firstLineIndex)+e.key+this.lines[lastLineNum].content.slice(lastLineIndex)
            
            let removed = this.lines.splice(delStart+1,delCount-1)
            lastLineNum -= removed.length

            this.focused = lastLineNum
            sel.removeAllRanges()
            this.$nextTick(()=>{
              this.focusLine(this.focused, firstLineIndex+1)
            })
          }
        }
      }
      catch {
        console.log('No focused text!')
      }
    })
    this.$refs.editor.addEventListener('keyup', (e)=>{
      try  {
        let selectedLines = this.findSelectedDivs(this.$refs.editor, window.getSelection().getRangeAt(0)).filter(e => e.classList.contains('line'))
        if(selectedLines.length==1) {
          if(e.key=='Enter') {
            e.preventDefault()
          }
          else if(e.key=='Backspace') {
            e.preventDefault()
            let line = this.getLineByIndex(this.focused)
            let lineContent = line.querySelector('.line-contents')
            this.lines[this.focused].content = lineContent.textContent

            const sel = window.getSelection();
            const node = sel.focusNode;
            const offset = sel.focusOffset;
            const pos = this.getCursorPosition(lineContent, node, offset, { pos: 0, done: false })

            this.$nextTick(()=>{
              this.focusLine(this.focused, pos.pos)
            })
          }
          else if(e.key=='Tab') {
            e.preventDefault()
          }
          else if(e.key.startsWith('Arrow')) {
            e.preventDefault()
          }
          else if(e.key.length === 1) {
            if(!e.ctrlKey) {
              e.preventDefault()
              let line = this.getLineByIndex(this.focused)
              let lineContent = line.getElementsByClassName('line-contents')[0]
              this.lines[this.focused].content = lineContent.textContent

              const sel = window.getSelection();
              const node = sel.focusNode;
              const offset = sel.focusOffset;
              const pos = this.getCursorPosition(lineContent, node, offset, { pos: 0, done: false })
              this.$nextTick(()=>{
                this.focusLine(this.focused, pos.pos)
              })
            }
          }
        }
        else if (selectedLines.length>1){
          e.preventDefault()
        }
      }
      catch {
        console.log('No focused text!')
      }
    })
  },
  watch: {
    text: {
      handler() {
        if(this.content!=this.text) {
          this.content = this.text
          this.lines = this.text.split('\n').map(l=>{return {'content': l.replace('\t','\u00A0\u00A0\u00A0\u00A0')}})
          this.$nextTick(()=>{
            this.focusLine(this.focused)
            this.$refs.editor.style.scrollBehavior = 'smooth'
            this.centerEditor()
            this.$refs.editor.style.scrollBehavior = 'auto'
          })
        }
      },
      deep: true
    },
    action: {
      handler() {
        if(this.action) {
          if(this.action.name=='update') {
            this.$emit('action', this.lines.map(l=>l.content).join('\n'))
          }
          else {
            console.warn('Unsupported action:', this.action.name);
            console.warn('Supported actions:\n'+['\t- update'].join('\n'))
          }
        }
      }
    }
  }
}
</script>

<style scoped>
#editor {
  height: calc(100%);
  width: calc(100%);
  box-sizing: border-box;
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  background: var(--background-alt);
}
.editor {
  height: calc(100%);
  width: calc(100%);
  outline: none;
  resize: none;
  color: var(--text);
  background: var(--background-alt);
  border: none;
  cursor: default;
  overflow-y: auto;
}
.editor::-webkit-scrollbar {
  width: 5px;
  background: var(--background-alt);
}
.editor::-webkit-scrollbar-thumb {
  background: var(--background-light);
}
.line {
  width: 100%;
  box-sizing: border-box;
  height: fit-content;
  display: flex;
  min-height: 20px;
}
.line-index {
  width: 30px;
  text-align: end;
  filter: brightness(.5);
  font-size: 14px;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: var(--font-title);
  cursor: default !important;
  user-select: none;
}
.line-contents {
  padding: 1px 3px 1px 10px;
  width: 100%;
  box-sizing: border-box;
  cursor: text;
  border-left: 1px solid var(--background);
}
.line:nth-last-child(1) {
  margin-bottom: 25%;
}
.focused .line-contents {
  color: var(--text-light);
  border-left: 1px solid var(--primary);
}
.focused .line-index {
  filter: brightness(1);
}
</style>
