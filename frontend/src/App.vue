<template>
  <div class="toolbar">
    <input type="file" @change="importContent" id="import" accept=".txt,.md" />
    <label  for="import" class="btn">
      <svg-icon type="mdi" :path="mdi.mdiExport" :size="22"></svg-icon>
    </label>

    <button class="btn" @click="exportContent()">
      <svg-icon type="mdi" :path="mdi.mdiImport" :size="22"></svg-icon>
    </button>
    <button class="btn" @click="updateContent()">
      <svg-icon type="mdi" :path="mdi.mdiPlayOutline" :size="22"></svg-icon>
    </button>
    <button class="btn" @click="viewParams=true">
      <svg-icon type="mdi" :path="mdi.mdiMenu" :size="22"></svg-icon>
    </button>
    <button class="btn" @click="viewActions=true">
      <svg-icon type="mdi" :path="mdi.mdiApi" :size="22"></svg-icon>
    </button>
  </div>
  <div id="main">
    <div class="message" v-if="message">
      {{ message }}
      <button class="btn" @click="message=''">
        <svg-icon type="mdi" :path="mdi.mdiClose" :size="16"></svg-icon>
      </button>
    </div>
    <div class="params popup" v-if="viewParams" >
      <h1>Params</h1>
      <div class="param">
          <label for="roundness">
            Roundness:
          </label>
        <input type="number" id="roundness" v-model.lazy="params.roundness"/>
      </div>
      <div class="param">
          <label for="roundness">
            Font Size:
          </label>
        <input type="number" id="fontSize" v-model.lazy="params.fontSize"/>
      </div>
      <div class="param">
          <label for="roundness">
            Text Font:
          </label>
        <input type="text" id="textFont" v-model.lazy="params.textFont"/>
      </div>
      <div class="param">
          <label for="roundness">
            Title Font:
          </label>
        <input type="text" id="titleFont" v-model.lazy="params.titleFont"/>
      </div>
      <button class="save" @click="viewParams=false">
        Save
      </button>
    </div>
    <div class="api popup" v-if="viewActions">
      <h1>API</h1>
      <button class="close" @click="viewActions=false">
        Close
      </button>
    </div>
    <Campus 
      :text="text"
      :params="params"
      :action="action"
      @action="response"
      />
  </div>
</template>

<script>
import { mdiPlayOutline, mdiImport, mdiExport, mdiMenu, mdiApi, mdiClose } from '@mdi/js';
import SvgIcon from '@jamescoyle/vue-icon';
import Campus from './components/Campus.vue'

export default {
  name: 'App',
  data() {
    return {
      text: "1\n2\n3\n4\n5\n6",
      params: {
        roundness: 5,
        fontSize: 14,
        textFont: "Segoe UI",
        titleFont: "Arial"
      },
      action: null,
      mdi: {
        mdiPlayOutline: mdiPlayOutline,
        mdiImport: mdiImport,
        mdiExport: mdiExport,
        mdiMenu: mdiMenu,
        mdiApi: mdiApi,
        mdiClose: mdiClose,
      },
      viewActions: false,
      viewParams: false,
      message: "",
    }
  },
  methods: {
    camelToDash(str) {
      return str.replace(/([A-Z])/g, (match) => `-${match.toLowerCase()}`);
    },
    exportContent() {
      this.updateContent()
      this.$nextTick(()=>{
        const mdData = this.text
        const blob = new Blob([mdData], {type: "text/md"})
        const url = URL.createObjectURL(blob)

        const a = document.createElement("a")
        a.href = url
        a.download = "file.md"
        a.click()
        
        URL.revokeObjectURL(url)
      })
    },
    importContent(event) {
      const file = event.target.files[0];
      if (!file) return;

      const reader = new FileReader();
      reader.onload = (e) => {
        const content = e.target.result;
        this.text = content
      };
      reader.readAsText(file);
    },
    updateContent() {
      this.action = {
        name : 'update'
      }
      this.message = "Text updated !"
      setTimeout(()=>{
        this.message = ""
      }, 3000)
    },
    response(response) {
      if(this.action.name == 'update') {
        this.text = response
        this.action = null
      }
      else {
        this.action = null
      }
    },
    loadParams() {
      document.documentElement.style.setProperty('--text-font', this.params.textFont);
      document.documentElement.style.setProperty('--title-font', this.params.titleFont);
      document.documentElement.style.setProperty('--font-size', this.params.fontSize+'px');
      document.documentElement.style.setProperty('--border-small', Math.ceil(this.params.roundness/2)+'px');
      document.documentElement.style.setProperty('--border-big', this.params.roundness+'px');
    }
  },
  mounted() {
    this.loadParams()
  },
  components: {
    Campus,
    SvgIcon
  },
  watch: {
    params: {
      handler() {
        this.loadParams()
      },
      deep: true
    }
  }
}
</script>

<style>
#app {
  height: 100vh;
  width: 100%;
  display: flex;
  flex-direction: column;
  font-size: var(--font-size);
}
.toolbar {
  height: 60px;
  background: var(--background);
  color: var(--text);
  display: flex;
  justify-content: flex-start;
  align-items: center;
  gap: 15px;
  padding-left: 20px;
  padding-right: 20px;
}
#main {
  font-family: var(--text-font);
  width: 100%;
  flex: 1;
  overflow: hidden;
  box-sizing: border-box;
  background: var(--background);
  position: relative;
  border-radius: 0px;
}
body, html {
  padding: 0;
  margin: 0;
  width: 100%;
  height: 100vh;
  background: var(--background);
  border: 1px solid var(--light);
}
:root {
  /* Primary */
  --primary: #4285F4;
  --primary-light: #82B1FF;
  --primary-dark: #0D47A1;

  /* Secondary */
  --secondary: #DB4437;
  --secondary-light: #FF7961;
  --secondary-dark: #B71C1C;

  /* Background */
  --background: #1b1b1b;
  --background-alt: #222222;
  --background-light: #3b3b3b;

  /* Text */
  --text-dark: #707070;
  --text-light: #ffffff;
  --text: #c7c7c7;

  /* Success */
  --success: #0F9D58;
  --success-light: #66BB6A;
  --success-dark: #1B5E20;

  /* Alert */
  --alert: #F4B400;
  --alert-light: #FFD54F;
  --alert-dark: #FF8F00;
}

.md-code {
  color: var(--text-light) !important;
  filter: brightness(50%);
  display: none;
}
.focused .md-code {
  display: inline;
}
.md-header-1 {
  font-weight: bold;
  font-size: 32px;
}
.md-header-2 {
  font-weight: bold;
  font-size: 26px;
}
.md-header-3 {
  font-weight: bold;
  font-size: 23px;
}
.md-header-4 {
  font-weight: bold;
  font-size: 20px;
}
.md-header-5 {
  font-weight: bold;
  font-size: 17px;
}
.md-header-6 {
  font-weight: bold;
  font-size: 15px;
}

input[type="file"] {
  display: none; 
}

.btn {
  cursor: pointer;
  width: 26px;
  height: 26px;
  background: none;
  border: none;
  color: var(--text);
  border-radius: 50%;
  padding: 0px;
  margin: 0px;
  position: relative;
}
.btn svg {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
.btn:hover {
  color: var(--primary);
}

.popup {
  position: absolute;
  background: var(--background);
  aspect-ratio: 9/14;
  width: 50%;
  max-width: 300px;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 2;
  color: var(--text);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  gap: 5px;
}
.popup input {
  background: transparent;
  background-color: none;
  border: 1px solid var(--background-light);
  border-radius: var(--border-small);
  width: 30px;
  color: var(--text);
  outline: none;
}
#textFont, #titleFont {
  width: 100px;
}

input[type=number]::-webkit-inner-spin-button {
  -webkit-appearance: none;
}
.param {
  width: 80%;
  display: flex;
  justify-content: space-between;
}
.save, .close {
  background: none;
  border: 1px solid var(--success-dark);
  border-radius: var(--border-small);
  position: absolute;
  bottom: 20px;
  right: 20px;
  color: var(--text);
  padding: 3px 8px 3px 8px;
  cursor: pointer;
}
.save:hover {
  background: var(--success);
}
.close {
  border: 1px solid var(--secondary-dark);
}
.close:hover {
  background: var(--secondary);
}

#app:has(.popup)::before {
  content: '';
  position: absolute;
  z-index: 1;
  height: 100%;
  width: 100%;
  top: 0px;
  left: 0px;
  backdrop-filter: blur(10px);
}
.message {
  position: absolute;
  top: 30px;
  left: 50%;
  transform: translateX(-50%);
  background: var(--background);
  border: 1px solid var(--primary);
  z-index: 1;
  animation: appear .3s ease-out;
  height: 35px;
  width: 150px;
  display: flex;
  justify-content: center;
  padding-right: 16px;
  align-items: center;
  color: var(--primary);
  user-select: none;
  border-radius: var(--border-big);
}
.message .btn {
  position: absolute;
  right: 5px;
  top: 50%;
  transform: translateY(-50%);
}

@keyframes appear {
  0% {
    top: 0px;
    opacity: 0;
  }
  30% {
    top: 20px;
    opacity: 1;
  }
  60% {
    top: 35px;
    opacity: 1;
  }
  100% {
    top: 30px;
    opacity: 1;
  }
}
</style>
