<template>
  <ph-window>
    <ph-toolbar type="header" title="Asar UI">
      <ph-toolbar-actions>
        <ph-button-group>
          <ph-button @click.native="open"><ph-icon icon="archive" :text="true"></ph-icon> Open</ph-button>
          <ph-button @click.native="extract" :disabled="!isOpen"><ph-icon icon="export" :text="true"></ph-icon> Extract</ph-button>
          <ph-button :disabled="true"><ph-icon icon="plus" :text="true"></ph-icon> New</ph-button>
        </ph-button-group>

        <ph-button @click.native="openAbout" class="pull-right">
          <ph-icon icon="newspaper" :text="true"></ph-icon> About
        </ph-button>
      </ph-toolbar-actions>
      <ph-toolbar-actions>
        <ph-button @click.native="back" :disabled="!history.length"><ph-icon icon="left" :text="true"></ph-icon> Back</ph-button>
      </ph-toolbar-actions>
    </ph-toolbar>
    <ph-window-content>
      <div id="dropZone" @click="open" v-show="!isOpen">
        <div class="box__input">
        <svg class="box__icon" xmlns="http://www.w3.org/2000/svg" width="50" height="43" viewBox="0 0 50 43"><path d="M48.4 26.5c-.9 0-1.7.7-1.7 1.7v11.6h-43.3v-11.6c0-.9-.7-1.7-1.7-1.7s-1.7.7-1.7 1.7v13.2c0 .9.7 1.7 1.7 1.7h46.7c.9 0 1.7-.7 1.7-1.7v-13.2c0-1-.7-1.7-1.7-1.7zm-24.5 6.1c.3.3.8.5 1.2.5.4 0 .9-.2 1.2-.5l10-11.6c.7-.7.7-1.7 0-2.4s-1.7-.7-2.4 0l-7.1 8.3v-25.3c0-.9-.7-1.7-1.7-1.7s-1.7.7-1.7 1.7v25.3l-7.1-8.3c-.7-.7-1.7-.7-2.4 0s-.7 1.7 0 2.4l10 11.6z"></path></svg>
        <label for="file"><strong>Choose a file</strong><span class="box__dragndrop"> or drag it here</span>.</label>
        </div>
      </div>
      
      <table class="table-striped" v-show="isOpen">
        <thead>
          <tr>
            <th style="width:43px">#</th>
            <th>Name</th>
            <th style="width:43px">Size</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item,key,index) in subFiles"
            :key="index"
            @dblclick="openIn(key,item)"
            @dragstart="rowDragStart($event,key)"
          >
            <td><ph-icon :icon="`${isDir(item)?'folder':'doc-text'}`"></ph-icon></td>
            <td>{{key}}</td>
            <td>{{`${isDir(item)?Object.keys(item.files).length:prettyBytes(item.size)}`}}</td>
          </tr>
        </tbody>
      </table>

    </ph-window-content>
    <ph-toolbar type="footer"/>
  </ph-window>
</template>

<script>
import path from 'path';
import fs from 'fs';
import asar from 'asar';
const prettyBytes = require('pretty-bytes');
import openAboutWindow from 'about-window';
import { ipcRenderer } from 'electron';

  let packFiles;

  export default {
    name: 'main-page',
    mounted(){
      document.ondragover = document.ondrop = (ev) => {
        ev.preventDefault()
      }

      document.getElementById('dropZone').ondrop = (ev) => {
        this.asarFileLocations = ev.dataTransfer.files[0].path;
        this.packName = this.asarFileLocations.split('/').pop();
        this.isOpen=false;
        this.history = [];
        this.packFiles = asar.listPackageWithHeader(this.asarFileLocations);
        this.subFiles=this.packFiles.files;
        this.isOpen = true;
        ev.preventDefault()
      }
    },
    data() {
      return {
        packName: null,
        isOpen: false,
        packFiles: {},
        subFiles: {},
        history: [],
        directory: [],
        asarFileLocations: null,
        tmpPath: null,
      };
    },
    methods: {
      rowDragStart(e,item){
        e.preventDefault();
        if(this.tmpPath == null) {
          this.tmpPath = path.join(this.$electron.remote.app.getPath('temp'),'asarui',this.makeid());
          asar.extractAll(this.asarFileLocations,this.tmpPath);
        }
        const filePath=path.join(this.tmpPath,...this.directory,item);
        ipcRenderer.send('ondragstart',filePath);
      },
      prettyBytes(b){
        return prettyBytes(b);
      },
      openAbout () {
        openAboutWindow({
          icon_path: path.join(__static,'asarui.png'),
          package_json_dir: path.resolve(),
          license: 'MIT',
          creator: 'Creator: myazarc (myazarc@gmail.com)',
          product_name: 'Asar UI',
          version: 'v1.0.0'
        });
      },
      open () {
        const file = this.$electron.remote.dialog.showOpenDialog({ properties: ['openFile'], filters: [
        { name: 'Asar Pack', extensions: ['asar'] },],
        });
        this.isOpen=false;
        if(file) {
          this.asarFileLocations = file[0];
          this.packName = this.asarFileLocations.split('/').pop();
          this.history = [];
          this.packFiles = asar.listPackageWithHeader(this.asarFileLocations);
          this.subFiles=this.packFiles.files;
          this.isOpen = true;
        }
      },
      extract(){
        const file = this.$electron.remote.dialog.showOpenDialog({ properties: ['openDirectory'] });
        asar.extractAll(this.asarFileLocations,path.join(file[0],this.packName));
      },
      back(){
        const data = this.history.pop();
        this.subFiles = data;
        this.directory.pop();
      },
      isDir(files){
        if(files.files) {
          return true;
        }
        return false;
      },
      openIn(key,item){
        if(this.isDir(item)){
          this.history.push(Object.assign({},this.subFiles));
          this.subFiles = this.subFiles[key].files;
          this.directory.push(key);
        }
      },
      makeid() {
        var text = "";
        var possible = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";

        for (var i = 0; i < 16; i++)
          text += possible.charAt(Math.floor(Math.random() * possible.length));

        return text;
      }
    },
    watch: {
      isOpen(val){
        if(!val){
          this.directory=[];
          this.tmpPath = null;
        }
      }
    },
  }
</script>

<style lang="scss" scoped>
.btn:disabled{
  cursor: not-allowed;
}

#dropZone{
  width: 100%;
  height: 100%;
  position: absolute;
  left: 0;
  top: 0;
  font-size: 1.25rem;
position: relative;
padding: 100px 20px;
cursor: pointer;

.box__input{
  text-align: center;
  cursor: pointer;

.box__icon {
    width: 100%;
    height: 80px;
    fill: #92b0b3;
    display: block;
    margin-bottom: 40px;
    cursor: pointer;

}

label {
    max-width: 80%;
    text-overflow: ellipsis;
    white-space: nowrap;
    cursor: pointer;
    display: inline-block;
    overflow: hidden;
}
}

}

table tbody tr {
  -webkit-user-drag: element;
}
</style>
