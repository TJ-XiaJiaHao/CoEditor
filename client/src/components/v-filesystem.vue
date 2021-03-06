<template>
  <div @click="leftClick" @contextmenu.prevent="rightClick" class="file-system" @scroll="scroll">
    <vFileTree :files="project.files" @loadFile="loadFile"></vFileTree>

    <div class="menu" v-if="menu.show" :style="{left: menu.left + 'px', top: menu.top + 'px', width: menu.width + 'px'}">
      <div class="menu-item" v-for="item in menu.items" :key="item.name" :style="{height: menu.itemHeight + 'px', lineHeight: menu.itemHeight + 'px'}" @click="handleFn(item.handle)">
        <img :src="item.img" class="menu-img"/>
        <label>{{item.name}}</label>
      </div>
    </div>

    <Modal
      width="200"
      :title="inputDialog.title"
      v-model="inputDialog.show"
      @on-ok="inputDialog.confirmHandle">
      <Input v-if="inputDialog.input !== null" v-model="inputDialog.input"  style="width: 150px" />
      <p v-if="inputDialog.msg !== null">{{inputDialog.msg}}</p>
    </Modal>
  </div>
</template>

<script>
import vFileTree from '@/components/v-filetree';
import directoryIcon from './../assets/directory.svg';
import fileIcon from './../assets/file.svg';
import deleteIcon from './../assets/delete.svg';
import renameIcon from './../assets/rename.svg';
import axios from 'axios';                                    // http协议库
export default {
  components: {
    vFileTree
  },
  props: {
    project: {
      type: Object,
      default: () => []
    },
    host: {
      type: String,
      default: ''
    }
  },
  data () {
    return {
      menu: {
        show: false,
        top: 0,
        left: 0,
        width: 100,
        itemHeight: 25,
        items: [{
          name: '新建文件夹',
          handle: 'createFolder',
          img: directoryIcon
        }, {
          name: '新建文件',
          handle: 'createFile',
          img: fileIcon
        }, {
          name: '删除',
          handle: 'delete',
          img: deleteIcon
        }, {
          name: '重命名',
          handle: 'rename',
          img: renameIcon
        }]
      },
      rightClickFile: null,
      scrollTop: 0,
      scrollLeft: 0,

      inputDialog: {
        show: false,
        title: '',
        input: '',
        msg: null,
        confirmHandle: '',
        cancelHandle: ''
      }
    };
  },
  methods: {
    handleFn (fn) {
      this.menu.show = false;
      this[fn]();
    },

    // 文件系统基本操作
    createFolder () {
      if (!this.project || !this.project.projectId) {
        this.showError('请选择一个项目');
        return;
      }
      this.showDialog('',  null, '请输入文件夹名', () => {
        axios.post(`${this.host}/projects/createFolder`, {
          projectId: this.project.projectId,
          folderName: this.inputDialog.input,
          fatherId: this.rightClickFile.id
        }).then((data) => {
          if (data.data.code !== 0)  this.showError('创建文件夹失败');
        });
      });
    },
    createFile () {
      if (!this.project || !this.project.projectId) {
        this.showError('请选择一个项目');
        return;
      }
      this.showDialog('',  null, '请输入文件名', () => {
        axios.post(`${this.host}/projects/createFile`, {
          projectId: this.project.projectId,
          fileName: this.inputDialog.input,
          fatherId: this.rightClickFile.id
        }).then((data) => {
          if (data.data.code !== 0) this.showError('创建文件失败');
        });
      });
    },
    delete () {
      if (this.rightClickFile.id === '-1') {
        this.showError('请选择要删除的文件');
        return;
      }
      this.showDialog(null, '删除后将无法回退，是否确认删除？', '确认框', () => {
        if (this.rightClickFile.type === 0) {
          axios.post(`${this.host}/projects/deleteFolder`, {
            projectId: this.project.projectId,
            folderId: this.rightClickFile.id
          }).then((data) => {
            if (data.data.code !== 0) this.showError('删除文件夹失败');
          });
        } else if (this.rightClickFile.type === 1) {
          axios.post(`${this.host}/projects/deleteFile`, {
            projectId: this.project.projectId,
            fileId: this.rightClickFile.id
          }).then((data) => {
            if (data.data.code !== 0) this.showError('删除文件失败');
          });
        }
      });
    },
    rename () {
      if (this.rightClickFile.id === '-1') {
        this.showError('请选择要重命名的文件');
        return;
      }
      if (this.rightClickFile.type === 0) {
        this.showDialog('',  null, '请输入文件夹名', () => {
          axios.post(`${this.host}/projects/renameFolder`, {
            projectId: this.project.projectId,
            folderId: this.rightClickFile.id,
            folderName: this.inputDialog.input
          }).then((data) => {
            if (data.data.code !== 0) this.showError('文件重命名失败');
          });
        });
      } else if (this.rightClickFile.type === 1) {
        this.showDialog('',  null, '请输入文件名', () => {
          axios.post(`${this.host}/projects/renameFile`, {
            projectId: this.project.projectId,
            fileId: this.rightClickFile.id,
            fileName: this.inputDialog.input
          }).then((data) => {
            if (data.data.code !== 0) this.showError('文件重命名失败');
          });
        });
      }
    },

    // 加载指定文件
    loadFile (id, filename) {
      this.$emit('loadFile', id, filename);
    },

    // 鼠标操作
    scroll (e) {
      this.scrollTop = e.target.scrollTop;
      this.scrollLeft = e.target.scrollLeft;
    },
    leftClick (e) {
      this.menu.show = false;
    },
    rightClick (e) {
      // 记录右击选中的文件或文件夹
      const folder = e.path.filter((item) => { return item.className && item.className.indexOf('type-folder') >= 0; })[0];
      const file = e.path.filter((item) => { return item.className && item.className.indexOf('type-file') >= 0; })[0];
      if (folder) this.rightClickFile = { type: 0, id: folder.id };
      else if (file) this.rightClickFile = { type: 1, id: file.id };
      else this.rightClickFile = {type: -1, id: '-1'};

      // 显示菜单栏
      const container = e.path.filter((item) => { return item.className === 'file-system'; })[0];
      const containerHeight = container.offsetHeight;
      const containerWidth = container.offsetWidth;
      this.menu.show = true;
      this.menu.top = e.clientY + this.scrollTop + 5 - 20;
      this.menu.left = e.clientX + this.scrollLeft + 5;
      if (e.clientX + this.menu.width > containerWidth) this.menu.left -= this.menu.width + 10;
      if (e.clientY + this.menu.items.length * this.menu.itemHeight > containerHeight) this.menu.top -= this.menu.items.length * this.menu.itemHeight + 10;
    },

    // 消息提示
    showDialog (input, msg, title, confirmHandle) {
      this.inputDialog.show = true;
      this.inputDialog.input = input;
      this.inputDialog.msg = msg;
      this.inputDialog.title = title;
      this.inputDialog.confirmHandle = confirmHandle;
    },
    showError (title, msg) {
      this.$Notice.error({
        title: title,
        desc: msg
      });
    },
    showSuccess (title, msg) {
      this.$Notice.success({
        title: title,
        desc: msg
      });
    }
  }
};
</script>
<style>
.file-system {
  width: 100%;
  height: 100%;
  overflow-x: hidden;
  overflow-y: scroll;
  position: relative;
}
.menu {
  position: absolute;
  border-radius: 2px;
  z-index: 1000;
  background: rgb(232, 232, 232);
  text-align: left;
}

.menu-item {
  font-size: 14px;
  cursor: default;
  padding-left: 5px;
}
.menu-item:hover {
  background: rgb(56, 117, 214);
  color: white;
}
.menu-img{
  width: 15px;
  height: 15px;
  vertical-align: center;
}
::-webkit-scrollbar {/*隐藏滚轮*/
  display: none;
}
</style>
