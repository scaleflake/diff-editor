<template>
  <div :class="s.wrapper">
    <div :class="s.menu">
      <PackageItem
        :package-name="currentPackageName"
      />
      <div :class="s.submenu" style="padding-top: 5px">
        <PackageItem
          v-for="(_, packageName) in stats.packages"
          :key="packageName"
          :package-name="packageName"
        />
      </div>
    </div>
    <div :class="s.editor" ref="editor" />
  </div>
</template>

<script>
import lo from 'lodash';
import Vue from 'vue';
import axios from 'axios';
import * as monaco from 'monaco-editor';

const PackageItem = Vue.extend({
  props: {
    packageName: String,
  },
  computed: {
    packageStats() {
      return this.$parent.getPackageStats(this.packageName);
    },
  },
  render() {
    const { packageName, packageStats } = this;
    const k = Math.ceil((packageStats.diffCount / packageStats.totalCount) * 1000) / 10;
    return (
      <div>
        <span
          style={{
            display: 'inline-block',
            width: '400px',
            cursor: 'pointer',
            color: packageName === this.$parent.currentPackageName ? '#66ffff' : 'unset',
          }}
          vOn:click={() => this.$parent.selectPackage(packageName)}
        >{packageName}</span>
        <span style="color: lightsteelblue">{` ${packageStats.totalCount}`}</span>
        <span style="color: orange">{` ${packageStats.diffCount}`}</span>
        <span style={{ color: k > 0 ? '#9d75cf' : '#aaaaaa' }}>
          {` (${k}%)`}
        </span>
      </div>
    );
  },
});

const baseUrl = 'http://localhost:8081';

export default {
  name: 'App',
  components: {
    PackageItem,
  },
  data() {
    return {
      stats: null,
    };
  },
  computed: {
    currentPackageName() {
      return window.location.pathname.replace('/', '');
    },
  },
  methods: {
    getPackageStats(packageName) {
      return this.stats?.packages[packageName] || {};
    },
    selectPackage(packageName) {
      window.location.href = `/${packageName}`;
    },
  },
  async mounted() {
    (async () => {
      const { data } = await axios.get(`${baseUrl}`);
      this.stats = data;
    })();

    const { currentPackageName: p } = this;

    const { data: packages } = await axios.get(`${baseUrl}/packages`);

    if (!packages.includes(p)) {
      this.selectPackage(packages[0]);
    }

    monaco.editor.defineTheme('myTheme', {
      base: 'vs-dark',
      inherit: true,
      rules: [],
      colors: {
        // #1e1e1e
        'diffEditor.insertedTextBackground': '#423324',
        'diffEditor.removedTextBackground': '#381e39',
      },
    });

    const diffEditor = monaco.editor.createDiffEditor(this.$refs.editor, {
      originalEditable: true,
      automaticLayout: true,
      theme: 'myTheme',
    });

    const { data } = await axios.get(`${baseUrl}/${p}`);
    const [pathA, codeA] = Object.entries(data['beautyagent-backend-shop'])[0];
    const [pathB, codeB] = Object.entries(data['beautyagent-backend-shop-parser'])[0];

    diffEditor.setModel({
      original: monaco.editor.createModel(codeA, pathA.endsWith('.js') ? 'javascript' : 'typescript'),
      modified: monaco.editor.createModel(codeB, pathB.endsWith('.js') ? 'javascript' : 'typescript'),
    });

    const editorA = diffEditor.getOriginalEditor();
    editorA.onDidChangeModelContent(lo.debounce(async () => {
      await axios.post(`${baseUrl}/${p}`, {
        'beautyagent-backend-shop': {
          [pathA]: editorA.getValue(),
        },
      });
    }, 300));

    const editorB = diffEditor.getModifiedEditor();
    editorB.onDidChangeModelContent(lo.debounce(async () => {
      await axios.post(`${baseUrl}/${p}`, {
        'beautyagent-backend-shop-parser': {
          [pathB]: editorB.getValue(),
        },
      });
    }, 300));
  },
};
</script>

<style module="s" lang="scss">
.wrapper {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-flow: column;
}
.menu {
  width: 100%;
  position: relative;
}
.submenu {
  position: absolute;
  z-index: 1;
  background: #101010;
  overflow: hidden;
  width: 100%;
  transform: scaleY(0);
  transform-origin: top;
  transition: transform 0.2s ease;
}
.menu:hover .submenu {
  transform: scaleY(1);
}

.editor {
  width: 100%;
  flex: 1;
}
</style>

<style>
body {
  margin: 0;
  background: #101010;
  color: #cecece;
  font-family: monospace;
  font-size: 15px;
}
</style>
