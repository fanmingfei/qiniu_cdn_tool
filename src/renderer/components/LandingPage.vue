<template>
    <div id="wrapper" ref="uploaderPleace">
        <div class="upload-box el-upload-dragger" v-if="list.length === 0">
            <i class="el-icon-upload"></i>
            <div class="el-upload__text">将文件拖到此处</div>
        </div>
        <div class="list" v-if="list.length > 0">
            <el-table :data="list" size="mini" :stripe="true" style="width: 100%" :row-class-name="tableRowClassName">
                <el-table-column>
                    <template slot-scope="scope">
                        {{getQualityUrl(scope.row.url)}}
                    </template>
                </el-table-column>
                <el-table-column width="280">
                    <template slot-scope="scope">
                        <el-popover placement="left" width="500" trigger="hover" content="" v-if="!scope.row.error">
                            <img :src="getQualityUrl(scope.row.url)" alt="" class="display-image" >
                            <el-button size="small" icon="el-icon-view" circle slot="reference" :disabled="!isImage(scope.row.url)"></el-button>
                        </el-popover>
                        <el-button @click="open(scope.row.url)" v-if="!scope.row.error" size="small" type="primary">打开链接</el-button>
                        <el-button @click="copy(scope.row.url)" v-if="!scope.row.error" size="small" type="success">复制链接</el-button>
                        <el-alert v-else title="上传失败" type="error" :closable="false">
                        </el-alert>
                    </template>
                </el-table-column>
            </el-table>
        </div>
        <el-row class="bottom">
            <el-col :span="2">
                <div class="upload-btn">
                    <input ref="input" type="file" multiple class="upload-input" @change="change">
                    <el-button type="primary" icon="el-icon-upload" circle></el-button>
                </div>
            </el-col>
            <el-col :span="10">
                <el-button type="info" icon="el-icon-setting" circle @click="showInfoDialog"></el-button>
            </el-col>
            <el-col :span="12">
                <el-row>
                    <el-col :span="9" style="font-size: 12px; line-height: 40px">
                        质量（不影响原图）
                    </el-col>
                    <el-col :span="15">
                        <el-slider v-model="quality"></el-slider>
                    </el-col>
                </el-row>
            </el-col>
        </el-row>
        <el-dialog ref="dialog" title="设置" :visible.sync="dialogVisible" width="30%">
            <el-input class="info-input" v-model="info.accessKey" placeholder="accessKey"></el-input>
            <el-input class="info-input" v-model="info.secretKey" placeholder="secretKey"></el-input>
            <el-input class="info-input" v-model="info.bucket" placeholder="bucket 仓库名称"></el-input>
            <el-input class="info-input" v-model="info.domain" placeholder="域名"></el-input>
            <el-input class="info-input" v-model="info.basePath" placeholder="路径前缀"></el-input>
            <div slot="footer" class="dialog-footer">
                <el-button type="primary" @click="setInfo">确 定</el-button>
            </div>
        </el-dialog>
    </div>
</template>
<script>
import fs from 'fs'
import path from 'path'
import { clipboard } from 'electron'
import streamBuffers from 'stream-buffers'

import qiniu from 'qiniu'
import md5 from 'md5'
import mime from 'mime'

export default {
    name: 'landing-page',
    components: {},
    data() {
        return {
            list: [],
            quality: 100,
            dialogVisible: false,
            info: {
                accessKey: localStorage.getItem('accessKey'),
                secretKey: localStorage.getItem('secretKey'),
                bucket: localStorage.getItem('bucket'),
                domain: localStorage.getItem('domain'),
                basePath: localStorage.getItem('basePath')
            }
        }
    },
    methods: {
        preventDefault(e) {
            console.log(e)
        },
        showInfoDialog() {
            this.dialogVisible = true
        },
        setInfo() {
            this.dialogVisible = false
            for (let key in this.info) {
                localStorage.setItem(key, this.info[key])
            }
            this.initQiniu()
        },
        tableRowClassName({ row, rowIndex }) {
            if (row.error) {
                return 'upload-err'
            }
        },
        open(url) {
            this.$electron.shell.openExternal(this.getQualityUrl(url))
        },
        initQiniu() {
            const mac = new qiniu.auth.digest.Mac(this.info.accessKey, this.info.secretKey);
            const options = {
                scope: this.info.bucket,
                expires: 7200
            };
            const putPolicy = new qiniu.rs.PutPolicy(options);
            this.uploadToken = putPolicy.uploadToken(mac);
            console.log(mac, putPolicy, this.uploadToken)
        },
        addItem(item) {
            console.log(item)
            this.list.push(item)
        },
        copy(url) {
            clipboard.writeText(this.getQualityUrl(url))
            this.$message({ message: '复制成功', type: 'success', duration: 500 });
        },
        getQualityUrl(url) {
            if (!this.isImage(url)) {
                return url
            }
            return this.quality === 100 ? url : url + '?imageView2/0/q/' + this.quality + '|imageslim'
        },
        isImage(url) {
            const m = mime.getType(url)
            return !(!m || m.indexOf('image') === -1)
        },
        change(e) {
            const files = this.$refs.input.files
            console.log(files)

            for (let f of files) {
                console.log('File(s) you selected here: ', f.path)
                this.upload(f.path)
            }
        },
        getName(content, ext) {
            const name = md5(content)
            return this.info.basePath + name + ext
        },
        getExt(filePath) {
            return path.parse(filePath).ext
        },
        upload(filePath) {
            const file = fs.readFileSync(filePath)
            const key = this.getName(file, this.getExt(filePath))
            const localFile = filePath
            const config = new qiniu.conf.Config();
            const formUploader = new qiniu.form_up.FormUploader(config);
            const putExtra = new qiniu.form_up.PutExtra();
            // 文件上传
            formUploader.putFile(this.uploadToken, key, localFile, putExtra, (respErr,
                respBody, respInfo) => {
                this.uploadCallback(respErr,
                    respBody, respInfo, localFile)
            });
        },
        initDrop() {

            const holder = this.$refs.uploaderPleace
            holder.ondragover = () => {
                return false;
            };

            holder.ondragleave = () => {
                return false;
            };

            holder.ondragend = () => {
                return false;
            };

            holder.ondrop = (e) => {
                e.preventDefault();

                for (let f of e.dataTransfer.files) {
                    console.log('File(s) you dragged here: ', f.path)
                    this.upload(f.path)
                }

                return false;
            };
        },
        uploadCallback(respErr,
            respBody, respInfo, localFile) {
            console.log(respErr,
                respBody, respInfo)
            let info
            if (respErr) {
                info = {
                    error: true,
                    url: localFile
                }
            } else if (respInfo.statusCode == 200) {
                info = {
                    url: this.info.domain + respBody.key
                }
            } else {
                info = {
                    error: true,
                    url: localFile
                }
            }
            this.addItem(info)
        },
        uploadStream() {
            const formats = clipboard.availableFormats()
            console.log(formats)
            if (formats.length !== 1) {
                this.$message({
                    type: 'info',
                    message: '目前只支持截图粘贴'
                })
                return
            }
            if (formats.join(',').indexOf('image') === -1) {
                this.$message({
                    type: 'info',
                    message: '目前只支持截图粘贴'
                })
                return
            }
            const img = clipboard.readImage()
            const jpg = img.toJPEG(100)
            const size = jpg.length * 8
            var stream = new streamBuffers.ReadableStreamBuffer({
                frequency: 10, // in milliseconds.
                chunkSize: size // in bytes.
            });
            stream.put(jpg)
            stream.stop()

            const key = this.getName(jpg, '.jpg')
            const config = new qiniu.conf.Config();
            const formUploader = new qiniu.form_up.FormUploader(config);
            const putExtra = new qiniu.form_up.PutExtra();
            const readableStream = stream; // 可读的流
            formUploader.putStream(this.uploadToken, key, readableStream, putExtra, (respErr,
                respBody, respInfo) => {
                this.uploadCallback(respErr,
                    respBody, respInfo, key)
            });
        },
        initCtrlV() {
            document.body.addEventListener('paste', e => {
                if (this.dialogVisible) { // worse
                    return
                }
                this.uploadStream()
            })
        },
        checkInfo() {
            const values = [this.info.accessKey, this.info.secretKey, this.info.bucket, this.info.domain]
            if (values.indexOf('') > -1) {
                this.showInfoDialog()
                return
            }
        }
    },
    mounted() {
        this.checkInfo()
        this.initQiniu()
        this.initDrop()
        this.initCtrlV()
    }
}
</script>
<style>
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

#wrapper {
    overflow: hidden;
    background:
        radial-gradient(ellipse at top left,
        rgba(255, 255, 255, 1) 40%,
        rgba(229, 229, 229, .9) 100%);
    height: 100vh;
    width: 100vw;
}

.list {
    position: absolute;
    overflow-y: scroll;
    top: 0;
    bottom: 60px;
    width: 100%;
}

.upload-box {
    cursor: default;
    margin: calc(50vh - 130px) auto;
}

.upload-btn {
    position: relative;
    width: 40px;
    height: 40px;
}

.upload-input {
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;
    opacity: 0;
}

.bottom {
    position: absolute;
    bottom: 0;
    width: 100vw;
    background: #ccc;
    padding: 10px 20px;
}

.display-image {
    width: calc(100% - 10px);
    height: calc(100% - 10px);
}

.upload-err {
    background: #ff9e00 !important;
}

.info-input {
    margin-bottom: 6px;
}
</style>