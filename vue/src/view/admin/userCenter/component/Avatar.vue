<template>
    <abstract-dialog :loading="loading" title="上传头像" :value="value" width="50%" @close="cancel">
        <div class="avatar-cropper">
            <vue-cropper
                ref="cropper"
                :img="img"
                :info="false"
                autoCrop
                autoCropHeight="200px"
                autoCropWidth="200px"
                fixedBox
                full
                outputType="png"
                @wheel.native.prevent="scale"
            />

            <input
                ref="input"
                accept="image/png, image/jpeg, image/gif, image/jpg"
                type="file"
                @change="chooseImage"
            >
        </div>

        <template v-slot:footer>
            <el-button plain size="small" @click="$refs.input.click()">选择图片</el-button>
            <el-button plain size="small" @click="closeDialog">取 消</el-button>
            <el-button size="small" type="primary" @click="confirm">确 定</el-button>
        </template>
    </abstract-dialog>
</template>

<script>
import {VueCropper} from 'vue-cropper'
import AbstractDialog from '@/component/abstract/Dialog'
import dialogMixin from "@/mixin/dialogMixin"
import {elError, elSuccess} from "@/util/message"
import {autoCompleteUrl, upload} from "@/util/file"
import {updateAvatar} from "@/api/account"

export default {
    name: "Avatar",

    mixins: [dialogMixin],

    components: {VueCropper, AbstractDialog},

    props: {
        value: Boolean
    },

    data() {
        return {
            loading: false,
            img: '',
            name: ''
        }
    },

    methods: {
        clear() {
            this.loading = false
            this.$refs.cropper.clearCrop()
            this.img && window.URL.revokeObjectURL(this.img)
            this.img = ''
            this.name = ''
        },

        scale(e) {
            const eventDelta = e.wheelDelta || -(e.detail || 0) * 40
            this.$refs.cropper.changeScale(eventDelta / 120)
        },

        chooseImage(e) {
            if (this.loading) return

            this.clear()

            const file = e.target.files[0]

            if (!file.type.includes('image')) {
                return elError('请上传图片')
            }

            if (file.size > 1048576) {
                return elError('上传的图片不能大于1M')
            }

            this.name = file.name
            const reader = new FileReader()
            reader.onload = e => {
                this.img = window.URL.createObjectURL(new Blob([e.target.result]))
            }
            reader.readAsArrayBuffer(file)
        },

        confirm() {
            if (!this.img) return elError('请先上传图片')
            if (this.loading) return
            this.loading = true
            this.$refs.cropper.getCropBlob(data => {
                upload(new Blob([data]), this.name)
                    .then(({data}) => updateAvatar.request(data.data))
                    .then(({data, msg}) => {
                        this.$store.commit('user/avatar', autoCompleteUrl(data))
                        this.$store.dispatch('user/refresh')
                        elSuccess(msg)
                    })
                    .finally(() => this.cancel())
            })
        },

        cancel() {
            this.closeDialog()
            this.clear()
        }
    }
}
</script>

<style lang="scss">
.avatar-cropper {
    height: 500px;
    width: 100%;
    border: 1px solid #ebebeb;

    .cropper-crop-box {
        border-radius: 50%;
        overflow: hidden;
    }

    > input[type=file] {
        display: none;
    }
}
</style>
