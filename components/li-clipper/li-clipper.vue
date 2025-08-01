<template>
    <view class="li-clipper" :class="{ open: value }" disable-scroll :style="'z-index: ' + zIndex + ';' + customStyle">
        <view class="li-clipper-mask" @touchstart.stop.prevent="clipTouchStart"
            @touchmove.stop.prevent="clipTouchMove" @touchend.stop.prevent="clipTouchEnd">
            <view class="li-clipper__content" :style="clipStyle">
                <view class="li-clipper__edge" v-for="(item, index) in [0, 0, 0, 0]" :key="index"></view>
            </view>
        </view>
        <image class="li-clipper-image" @error="imageLoad" @load="imageLoad" @touchstart="imageTouchStart"
            @touchmove="imageTouchMove" @touchend="imageTouchEnd" :src="image"
            :mode="imageWidth == 'auto' ? 'widthFix' : 'scaleToFill'" v-if="image" :style="imageStyle" />
        <canvas canvas-id="li-clipper" id="li-clipper" disable-scroll
            :style="'width: ' + canvasWidth * scaleRatio + 'px; height:' + canvasHeight * scaleRatio + 'px;'"
            class="li-clipper-canvas"></canvas>
        <view class="li-clipper-tools">
            <view class="li-clipper-tools__btns">
                <view v-if="isShowCancelBtn" @tap="cancel">
                    <slot name="cancel" v-if="$slots.cancel" />
                    <view v-else class="cancel">取消</view>
                </view>
                <view v-if="isShowPhotoBtn" @tap="uploadImage">
                    <slot name="photo" v-if="$slots.photo" />
                    <image v-else src="/static/photo.svg" />
                </view>
                <view v-if="isShowRotateBtn" @tap="rotate">
                    <slot name="rotate" v-if="$slots.rotate" />
                    <image v-else src="/static/rotate.svg" data-type="inverse" />
                </view>
                <view v-if="isShowConfirmBtn" @tap="confirm">
                    <slot name="confirm" v-if="$slots.confirm" />
                    <view v-else class="confirm">确定</view>
                </view>
            </view>
            <slot></slot>
        </view>
    </view>
</template>

<script>
import {
    determineDirection,
    calcImageOffset,
    calcImageScale,
    calcImageSize,
    calcPythagoreanTheorem,
    clipTouchMoveOfCalculate,
    imageTouchMoveOfCalcOffset,
} from './utils';
const cache = {}
export default {
    name: 'li-clipper',
    props: {
        value: {
            type: Boolean,
            default: true
        },
        // #ifdef MP-WEIXIN
        type: {
            type: String,
            default: '2d'
        },
        // #endif
        customStyle: {
            type: String,
        },
        zIndex: {
            type: Number,
            default: 99
        },
        imageUrl: {
            type: String
        },
        fileType: {
            type: String,
            default: 'png'
        },
        quality: {
            type: Number,
            default: 1
        },
        width: {
            type: Number,
            default: 400
        },
        height: {
            type: Number,
            default: 400
        },
        minWidth: {
            type: Number,
            default: 200
        },
        maxWidth: {
            type: Number,
            default: 600
        },
        destWidth: Number,
        destHeight: Number,
        minHeight: {
            type: Number,
            default: 200
        },
        maxHeight: {
            type: Number,
            default: 600
        },
        isLockWidth: {
            type: Boolean,
            default: false
        },
        isLockHeight: {
            type: Boolean,
            default: false
        },
        isLockRatio: {
            type: Boolean,
            default: true
        },
        scaleRatio: {
            type: Number,
            default: 1
        },
        minRatio: {
            type: Number,
            default: 0.5
        },
        maxRatio: {
            type: Number,
            default: 2
        },
        isDisableScale: {
            type: Boolean,
            default: false
        },
        isDisableRotate: {
            type: Boolean,
            default: false
        },
        isLimitMove: {
            type: Boolean,
            default: false
        },
        isShowPhotoBtn: {
            type: Boolean,
            default: true
        },
        isShowRotateBtn: {
            type: Boolean,
            default: true
        },
        isShowConfirmBtn: {
            type: Boolean,
            default: true
        },
        isShowCancelBtn: {
            type: Boolean,
            default: true
        },
        rotateAngle: {
            type: Number,
            default: 90
        },
        source: {
            type: Object,
            default: () => ({
                album: '从相册中选择',
                camera: '拍照',
                // #ifdef MP-WEIXIN
                message: '从微信中选择'
                // #endif
            })
        }
    },
    data() {
        return {
            canvasWidth: 0,
            canvasHeight: 0,
            clipX: 0,
            clipY: 0,
            clipWidth: 0,
            clipHeight: 0,
            animation: false,
            imageWidth: 0,
            imageHeight: 0,
            imageTop: 0,
            imageLeft: 0,
            scale: 1,
            angle: 0,
            image: '',
            imageInit: false,
            sysinfo: {},
            throttleTimer: null,
            throttleFlag: true,
            timeClipCenter: null,
            flagClipTouch: false,
            flagEndTouch: false,
            clipStart: {},
            animationTimer: null,
            touchRelative: [{ x: 0, y: 0 }],
            hypotenuseLength: 0,
            ctx: null,
            canvasId: 'li-clipper'
        };
    },
    computed: {
        // canvasId() {
        // 	return `l-clipper-${this._ ? this._.uid : this._uid}`
        // },
        clipStyle() {
            const { clipWidth, clipHeight, clipY, clipX, animation } = this
            return `
			width: ${clipWidth}px;
			height:${clipHeight}px;
			transition-property: ${animation ? '' : 'background'};
			left: ${clipX}px;
			top: ${clipY}px
			`
        },
        imageStyle() {
            const { imageWidth, imageHeight, imageLeft, imageTop, animation, scale, angle } = this
            return `
				width: ${imageWidth ? imageWidth + 'px' : 'auto'};
				height: ${imageHeight ? imageHeight + 'px' : 'auto'};
				transform: translate3d(${imageLeft - imageWidth / 2}px, ${imageTop - imageHeight / 2}px, 0) scale(${scale}) rotate(${angle}deg);
				transition-duration: ${animation ? 0.35 : 0}s
			`
        },
        clipSize() {
            const { clipWidth, clipHeight } = this;
            return { clipWidth, clipHeight };
        },
        clipPoint() {
            const { clipY, clipX } = this;
            return { clipY, clipX };
        }
    },
    watch: {
        value(val) {
            if (!val) {
                this.animation = 0
                this.angle = 0
            } else {
                if (this.imageUrl) {
                    const { imageWidth, imageHeight, imageLeft, imageTop, scale, clipX, clipY, clipWidth, clipHeight, path } = cache?.[this.imageUrl] || {}
                    if (path != this.image) {
                        this.image = this.imageUrl;
                    } else {
                        this.setDiffData({ imageWidth, imageHeight, imageLeft, imageTop, scale, clipX, clipY, clipWidth, clipHeight })
                    }

                }

            }
        },
        imageUrl(url) {
            this.image = url
        },
        image: {
            handler: async function (url) {
                this.getImageInfo(url)
            },
            // immediate: true,
        },
        clipSize({ widthVal, heightVal }) {
            let { minWidth, minHeight } = this;
            minWidth = minWidth / 2;
            minHeight = minHeight / 2;
            if (widthVal < minWidth) {
                this.setDiffData({ clipWidth: minWidth })
            }
            if (heightVal < minHeight) {
                this.setDiffData({ clipHeight: minHeight })
            }
            this.calcClipSize();
        },
        angle(val) {
            this.animation = this.imageInit;
            this.moveStop();
            const { isLimitMove } = this;
            if (isLimitMove && val % 90) {
                this.setDiffData({
                    angle: Math.round(val / 90) * 90
                })
            }
            this.imgMarginDetectionScale();
        },
        animation(val) {
            clearTimeout(this.animationTimer);
            if (val) {
                let animationTimer = setTimeout(() => {
                    this.setDiffData({
                        animation: false
                    })
                }, 260);
                this.setDiffData({ animationTimer })
                this.animationTimer = animationTimer;
            }
        },
        isLimitMove(val) {
            if (val) {
                if (this.angle % 90) {
                    this.setDiffData({
                        angle: Math.round(this.angle / 90) * 90
                    })
                }
                this.imgMarginDetectionScale();
            }
        },
        clipPoint() {
            this.cutDetectionPosition();
        },
        width(width, oWidth) {
            if (width !== oWidth) {
                this.setDiffData({
                    clipWidth: uni.upx2px(width) //width / 2
                })
            }
        },
        height(height, oHeight) {
            if (height !== oHeight) {
                this.setDiffData({
                    clipHeight: uni.upx2px(height) //height / 2
                })
            }
        }
    },
    mounted() {
        const sysinfo = uni.getSystemInfoSync();
        this.sysinfo = sysinfo;
        this.setClipInfo();
        this.image = this.imageUrl || this.image
        this.setClipCenter();
        this.calcClipSize();
        this.cutDetectionPosition();
    },
    methods: {
        setDiffData(data) {
            Object.keys(data).forEach(key => {
                if (this[key] !== data[key]) {
                    this[key] = data[key];
                }
            });
        },
        getImageInfo(url) {
            if (!url) return;
            if (this.value) {
                uni.showLoading({
                    title: '请稍候...',
                    mask: true
                });
            }
            this.imageInit = false
            uni.getImageInfo({
                src: url,
                success: res => {
                    if (['right', 'left'].includes(res.orientation)) {
                        this.imgComputeSize(res.height, res.width);
                    } else {
                        this.imgComputeSize(res.width, res.height);
                    }
                    this.image = res.path;
                    if (this.isLimitMove) {
                        this.imgMarginDetectionScale();
                        this.$emit('ready', res);
                    }
                    const { imageWidth, imageHeight, imageLeft, imageTop, scale, clipX, clipY, clipWidth, clipHeight } = this
                    cache[url] = Object.assign(res, { imageWidth, imageHeight, imageLeft, imageTop, scale, clipX, clipY, clipWidth, clipHeight });
                },
                fail: (err) => {
                    this.imgComputeSize();
                    if (this.isLimitMove) {
                        this.imgMarginDetectionScale();
                    }
                }
            });

        },
        setClipInfo() {
            const { width, height, sysinfo, canvasId } = this;
            const clipWidth = width / 2;
            const clipHeight = height / 2;
            const clipY = (sysinfo.windowHeight - clipHeight) / 2;
            const clipX = (sysinfo.windowWidth - clipWidth) / 2;
            const imageLeft = sysinfo.windowWidth / 2;
            const imageTop = sysinfo.windowHeight / 2;
            this.ctx = uni.createCanvasContext(canvasId, this);
            this.clipWidth = clipWidth;
            this.clipHeight = clipHeight;
            this.clipX = clipX;
            this.clipY = clipY;
            this.canvasHeight = clipHeight;
            this.canvasWidth = clipWidth;
            this.imageLeft = imageLeft;
            this.imageTop = imageTop;
        },
        setClipCenter() {
            const { sysInfo, clipHeight, clipWidth, imageTop, imageLeft } = this;
            let sys = sysInfo || uni.getSystemInfoSync();
            let clipY = (sys.windowHeight - clipHeight) * 0.5;
            let clipX = (sys.windowWidth - clipWidth) * 0.5;
            this.imageTop = imageTop - this.clipY + clipY;
            this.imageLeft = imageLeft - this.clipX + clipX;
            this.clipY = clipY;
            this.clipX = clipX;
        },
        calcClipSize() {
            const { clipHeight, clipWidth, sysinfo, clipX, clipY } = this;
            if (clipWidth > sysinfo.windowWidth) {
                this.setDiffData({
                    clipWidth: sysinfo.windowWidth
                })
            } else if (clipWidth + clipX > sysinfo.windowWidth) {
                this.setDiffData({
                    clipX: sysinfo.windowWidth - clipX
                })
            }
            if (clipHeight > sysinfo.windowHeight) {
                this.setDiffData({
                    clipHeight: sysinfo.windowHeight
                })
            } else if (clipHeight + clipY > sysinfo.windowHeight) {
                this.clipY = sysinfo.windowHeight - clipY;
                this.setDiffData({
                    clipY: sysinfo.windowHeight - clipY
                })
            }
        },
        cutDetectionPosition() {
            const { clipX, clipY, sysinfo, clipHeight, clipWidth } = this;
            let cutDetectionPositionTop = () => {
                if (clipY < 0) {
                    this.setDiffData({ clipY: 0 })
                }
                if (clipY > sysinfo.windowHeight - clipHeight) {
                    this.setDiffData({ clipY: sysinfo.windowHeight - clipHeight })
                }
            },
                cutDetectionPositionLeft = () => {
                    if (clipX < 0) {
                        this.setDiffData({ clipX: 0 })
                    }
                    if (clipX > sysinfo.windowWidth - clipWidth) {
                        this.setDiffData({ clipX: sysinfo.windowWidth - clipWidth })
                    }
                };
            if (clipY === null && clipX === null) {
                let newClipY = (sysinfo.windowHeight - clipHeight) * 0.5;
                let newClipX = (sysinfo.windowWidth - clipWidth) * 0.5;
                this.setDiffData({
                    clipX: newClipX,
                    clipY: newClipY
                })
            } else if (clipY !== null && clipX !== null) {
                cutDetectionPositionTop();
                cutDetectionPositionLeft();
            } else if (clipY !== null && clipX === null) {
                cutDetectionPositionTop();
                this.setDiffData({
                    clipX: (sysinfo.windowWidth - clipWidth) / 2
                })
            } else if (clipY === null && clipX !== null) {
                cutDetectionPositionLeft();
                this.setDiffData({
                    clipY: (sysinfo.windowHeight - clipHeight) / 2
                })
            }
        },
        imgComputeSize(width, height) {
            const { imageWidth, imageHeight } = calcImageSize(width, height, this);
            this.imageWidth = imageWidth;
            this.imageHeight = imageHeight;
        },
        imgMarginDetectionScale(scale) {
            if (!this.isLimitMove) return;
            const currentScale = calcImageScale(this, scale);
            this.imgMarginDetectionPosition(currentScale);
        },
        imgMarginDetectionPosition(scale) {
            if (!this.isLimitMove) return;
            const { scale: currentScale, left, top } = calcImageOffset(this, scale);
            this.setDiffData({
                imageLeft: left,
                imageTop: top,
                scale: currentScale
            })
        },
        throttle() {
            this.setDiffData({
                throttleFlag: true
            })
        },
        moveDuring() {
            clearTimeout(this.timeClipCenter);
        },
        moveStop() {
            clearTimeout(this.timeClipCenter);
            const timeClipCenter = setTimeout(() => {
                if (!this.animation) {
                    this.setDiffData({
                        imageInit: true,
                        animation: true,
                    })
                }
                this.setClipCenter();
            }, 800);
            this.setDiffData({ timeClipCenter })
        },
        clipTouchStart(event) {
            // #ifdef H5
            event.preventDefault()
            // #endif
            if (!this.image) {
                uni.showToast({
                    title: '请选择图片',
                    icon: 'none'
                });
                return;
            }
            const currentX = event.touches[0].clientX;
            const currentY = event.touches[0].clientY;
            const { clipX, clipY, clipWidth, clipHeight } = this;
            const corner = determineDirection(clipX, clipY, clipWidth, clipHeight, currentX, currentY);
            this.moveDuring();
            if (!corner) { return }
            this.clipStart = {
                width: clipWidth,
                height: clipHeight,
                x: currentX,
                y: currentY,
                clipY,
                clipX,
                corner
            };
            this.flagClipTouch = true;
            this.flagEndTouch = true;
        },
        clipTouchMove(event) {
            // #ifdef H5
            event.stopPropagation()
            event.preventDefault()
            // #endif
            if (!this.image) {
                uni.showToast({
                    title: '请选择图片',
                    icon: 'none'
                });
                return;
            }
            // 只针对单指点击做处理
            if (event.touches.length !== 1) {
                return;

            }
            const { flagClipTouch, throttleFlag } = this;
            if (flagClipTouch && throttleFlag) {
                const { isLockRatio, isLockHeight, isLockWidth } = this;
                if (isLockRatio && (isLockWidth || isLockHeight)) return;
                this.setDiffData({
                    throttleFlag: false
                })
                this.throttle();
                const clipData = clipTouchMoveOfCalculate(this, event);
                if (clipData) {
                    const { width, height, clipX, clipY } = clipData;
                    if (!isLockWidth && !isLockHeight) {
                        this.setDiffData({
                            clipWidth: width,
                            clipHeight: height,
                            clipX,
                            clipY
                        })
                    } else if (!isLockWidth) {
                        this.setDiffData({
                            clipWidth: width,
                            clipX
                        })
                    } else if (!isLockHeight) {
                        this.setDiffData({
                            clipHeight: height,
                            clipY
                        })
                    }
                    this.imgMarginDetectionScale();
                }

            }
        },
        clipTouchEnd() {
            this.moveStop();
            this.flagClipTouch = false;
        },
        imageTouchStart(e) {
            // #ifdef H5
            event.preventDefault()
            // #endif
            this.flagEndTouch = false;
            const { imageLeft, imageTop } = this;
            const clientXForLeft = e.touches[0].clientX;
            const clientYForLeft = e.touches[0].clientY;

            let touchRelative = [];
            if (e.touches.length === 1) {
                touchRelative[0] = {
                    x: clientXForLeft - imageLeft,
                    y: clientYForLeft - imageTop
                };
                this.touchRelative = touchRelative;
            } else {
                const clientXForRight = e.touches[1].clientX;
                const clientYForRight = e.touches[1].clientY;
                let width = Math.abs(clientXForLeft - clientXForRight);
                let height = Math.abs(clientYForLeft - clientYForRight);
                const hypotenuseLength = calcPythagoreanTheorem(width, height);

                touchRelative = [
                    {
                        x: clientXForLeft - imageLeft,
                        y: clientYForLeft - imageTop
                    },
                    {
                        x: clientXForRight - imageLeft,
                        y: clientYForRight - imageTop
                    }
                ];
                this.touchRelative = touchRelative;
                this.hypotenuseLength = hypotenuseLength;
            }
        },
        imageTouchMove(e) {
            // #ifdef H5
            event.preventDefault()
            // #endif
            const { flagEndTouch, throttleFlag } = this;
            if (flagEndTouch || !throttleFlag) return;
            const clientXForLeft = e.touches[0].clientX;
            const clientYForLeft = e.touches[0].clientY;
            this.setDiffData({ throttleFlag: false })
            this.throttle();
            this.moveDuring();
            if (e.touches.length === 1) {
                const { left: imageLeft, top: imageTop } = imageTouchMoveOfCalcOffset(this, clientXForLeft, clientYForLeft);
                this.setDiffData({
                    imageLeft,
                    imageTop
                })
                this.imgMarginDetectionPosition();
            } else {
                const clientXForRight = e.touches[1].clientX;
                const clientYForRight = e.touches[1].clientY;
                let width = Math.abs(clientXForLeft - clientXForRight),
                    height = Math.abs(clientYForLeft - clientYForRight),
                    hypotenuse = calcPythagoreanTheorem(width, height),
                    scale = this.scale * (hypotenuse / this.hypotenuseLength);
                if (this.isDisableScale) {

                    scale = 1;
                } else {
                    scale = scale <= this.minRatio ? this.minRatio : scale;
                    scale = scale >= this.maxRatio ? this.maxRatio : scale;
                    this.$emit('change', {
                        width: this.imageWidth * scale,
                        height: this.imageHeight * scale
                    });
                }

                this.imgMarginDetectionScale(scale);
                this.hypotenuseLength = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2));
                this.scale = scale;
            }
        },
        imageTouchEnd() {
            this.setDiffData({
                flagEndTouch: true
            })
            this.moveStop();
        },
        uploadImage() {
            const itemList = Object.entries(this.source)
            const sizeType = ['original', 'compressed']
            const success = ({ tempFilePaths: a, tempFiles: b }) => {
                this.image = a ? a[0] : b[0].path
            };
            const _uploadImage = (type) => {
                if (type !== 'message') {
                    uni.chooseImage({
                        count: 1,
                        sizeType,
                        sourceType: [type],
                        success
                    });
                }
                // #ifdef MP-WEIXIN
                if (type == 'message') {
                    wx.chooseMessageFile({
                        count: 1,
                        type: 'image',
                        success
                    })
                }
                // #endif
            }
            if (itemList.length > 1) {
                uni.showActionSheet({
                    itemList: itemList.map(v => v[1]),
                    success: ({ tapIndex: i }) => {
                        _uploadImage(itemList[i][0])
                    }
                })
            } else {
                _uploadImage(itemList[0][0])
            }
        },
        imageReset() {
            const sys = this.sysinfo || uni.getSystemInfoSync();
            this.moveStop()
            this.scale = 1;
            this.angle = 0;

            this.imageTop = sys.windowHeight / 2;
            this.imageLeft = sys.windowWidth / 2;
        },
        imageLoad(e) {
            this.imageReset();
            uni.hideLoading();
            this.$emit('ready', e.detail);
        },
        rotate(event) {
            if (this.isDisableRotate) return;
            if (!this.image) {
                uni.showToast({
                    title: '请选择图片',
                    icon: 'none'
                });
                return;
            }
            const { rotateAngle } = this;
            const originAngle = this.angle
            const type = event.currentTarget.dataset.type;
            if (type === 'along') {
                this.angle = originAngle + rotateAngle
            } else {
                this.angle = originAngle - rotateAngle
            }
            this.$emit('rotate', this.angle);
        },
        dataURLtoBlob(dataurl) {
            var arr = dataurl.split(','), mime = arr[0].match(/:(.*?);/)[1],
                bstr = atob(arr[1]), n = bstr.length, u8arr = new Uint8Array(n);
            while (n--) {
                u8arr[n] = bstr.charCodeAt(n);
            }
            return new Blob([u8arr], { type: mime });
        },
        confirm() {
            if (!this.image) {
                uni.showToast({ title: '请选择图片', icon: 'none' });
                return;
            }
            uni.showLoading({ title: '加载中' });
            
            // 统一的裁剪逻辑 - 实现真正的"所见即所得"
            const performCrop = (imgInfo) => {
                // 1. 获取原始图片尺寸
                const originalWidth = imgInfo.width || imgInfo.naturalWidth;
                const originalHeight = imgInfo.height || imgInfo.naturalHeight;
                
                // 2. 计算图片在屏幕上的实际显示尺寸和位置
                const displayWidth = this.imageWidth * this.scale;
                const displayHeight = this.imageHeight * this.scale;
                const imageLeft = this.imageLeft - displayWidth / 2;
                const imageTop = this.imageTop - displayHeight / 2;
                
                // 3. 计算裁剪框相对于图片显示区域的位置
                const relativeX = this.clipX - imageLeft;
                const relativeY = this.clipY - imageTop;
                
                // 4. 计算原图与显示图片的比例
                const scaleX = originalWidth / displayWidth;
                const scaleY = originalHeight / displayHeight;
                
                // 5. 计算旋转角度
                const angle = this.angle || 0;
                const angleRad = angle * Math.PI / 180;
                
                // 6. 计算图片中心点
                const imageCenterX = displayWidth / 2;
                const imageCenterY = displayHeight / 2;
                
                // 7. 计算裁剪框左上角和右下角在显示图像上的坐标
                const clipLeftTopX = relativeX;
                const clipLeftTopY = relativeY;
                const clipRightBottomX = relativeX + this.clipWidth;
                const clipRightBottomY = relativeY + this.clipHeight;
                
                // 8. 以图片中心为原点，做逆向旋转
                function rotatePoint(x, y, cx, cy, angleRad) {
                    const dx = x - cx;
                    const dy = y - cy;
                    const rx = dx * Math.cos(-angleRad) - dy * Math.sin(-angleRad);
                    const ry = dx * Math.sin(-angleRad) + dy * Math.cos(-angleRad);
                    return [cx + rx, cy + ry];
                }
                const [oriLeftTopX, oriLeftTopY] = rotatePoint(clipLeftTopX, clipLeftTopY, imageCenterX, imageCenterY, angleRad);
                const [oriRightBottomX, oriRightBottomY] = rotatePoint(clipRightBottomX, clipRightBottomY, imageCenterX, imageCenterY, angleRad);
                
                // 9. 转换到原图坐标
                const cropX = Math.round(Math.min(oriLeftTopX, oriRightBottomX) * scaleX);
                const cropY = Math.round(Math.min(oriLeftTopY, oriRightBottomY) * scaleY);
                const cropW = Math.round(Math.abs(oriRightBottomX - oriLeftTopX) * scaleX);
                const cropH = Math.round(Math.abs(oriRightBottomY - oriLeftTopY) * scaleY);
                
                // 10. 边界检查
                const safeCropX = Math.max(0, Math.min(cropX, originalWidth - 1));
                const safeCropY = Math.max(0, Math.min(cropY, originalHeight - 1));
                const safeCropW = Math.max(1, Math.min(cropW, originalWidth - safeCropX));
                const safeCropH = Math.max(1, Math.min(cropH, originalHeight - safeCropY));
                
                // 11. 计算旋转后的输出尺寸
                const cos = Math.abs(Math.cos(angleRad));
                const sin = Math.abs(Math.sin(angleRad));
                const outputWidth = Math.round(safeCropW * cos + safeCropH * sin);
                const outputHeight = Math.round(safeCropW * sin + safeCropH * cos);
                
                return {
                    cropX: safeCropX, cropY: safeCropY, cropW: safeCropW, cropH: safeCropH,
                    outputWidth, outputHeight,
                    angleRad, originalWidth, originalHeight,
                    imgPath: imgInfo.path || this.image
                };
            };
            
            // #ifdef MP-WEIXIN
            uni.getImageInfo({
                src: this.image,
                success: (imgInfo) => {
                    const cropData = performCrop(imgInfo);
                    
                    this.canvasWidth = cropData.outputWidth;
                    this.canvasHeight = cropData.outputHeight;
                    
                    this.$nextTick(() => {
                        const ctx = uni.createCanvasContext('li-clipper', this);
                        ctx.clearRect(0, 0, cropData.outputWidth, cropData.outputHeight);
                        
                        // 应用旋转变换
                        ctx.save();
                        ctx.translate(cropData.outputWidth / 2, cropData.outputHeight / 2);
                        ctx.rotate(cropData.angleRad);
                        ctx.drawImage(
                            cropData.imgPath,
                            cropData.cropX, cropData.cropY, cropData.cropW, cropData.cropH,
                            -cropData.cropW / 2, -cropData.cropH / 2, cropData.cropW, cropData.cropH
                        );
                        ctx.restore();
                        
                        ctx.draw(false, () => {
                            const MAX_SIDE = 1200;
                            let exportWidth = cropData.outputWidth;
                            let exportHeight = cropData.outputHeight;
                            if (cropData.outputWidth > MAX_SIDE || cropData.outputHeight > MAX_SIDE) {
                                const scale = MAX_SIDE / Math.max(cropData.outputWidth, cropData.outputHeight);
                                exportWidth = Math.round(cropData.outputWidth * scale);
                                exportHeight = Math.round(cropData.outputHeight * scale);
                            }
                            uni.canvasToTempFilePath({
                                x: 0,
                                y: 0,
                                width: cropData.outputWidth,
                                height: cropData.outputHeight,
                                destWidth: exportWidth,
                                destHeight: exportHeight,
                                canvasId: 'li-clipper',
                                fileType: this.fileType,
                                quality: this.quality,
                                success: (res) => {
                                    uni.hideLoading();
                                    this.$emit('success', {
                                        url: res.tempFilePath,
                                        width: exportWidth,
                                        height: exportHeight
                                    });
                                    this.$emit('input', false)
                                },
                                fail: (error) => {
                                    uni.hideLoading();
                                    this.$emit('fail', error);
                                    this.$emit('input', false)
                                }
                            }, this);
                        });
                    });
                },
                fail: () => {
                    uni.hideLoading();
                    uni.showToast({ title: '图片信息获取失败', icon: 'none' });
                }
            });
            // #endif

            // #ifdef H5
            setTimeout(() => {
                const img = new window.Image();
                img.src = this.image;
                img.onload = () => {
                    const cropData = performCrop(img);
                    
                    // 创建临时canvas进行裁剪
                    const tempCanvas = document.createElement('canvas');
                    tempCanvas.width = cropData.cropW;
                    tempCanvas.height = cropData.cropH;
                    const tempCtx = tempCanvas.getContext('2d');
                    
                    // 从原图中裁剪出指定区域
                    tempCtx.drawImage(
                        img,
                        cropData.cropX, cropData.cropY, cropData.cropW, cropData.cropH,
                        0, 0, cropData.cropW, cropData.cropH
                    );
                    
                    // 创建最终canvas，应用旋转
                    const finalCanvas = document.createElement('canvas');
                    const finalCtx = finalCanvas.getContext('2d');
                    
                    finalCanvas.width = cropData.outputWidth;
                    finalCanvas.height = cropData.outputHeight;
                    
                    // 应用旋转变换
                    finalCtx.save();
                    finalCtx.translate(cropData.outputWidth / 2, cropData.outputHeight / 2);
                    finalCtx.rotate(cropData.angleRad);
                    finalCtx.drawImage(
                        tempCanvas,
                        -cropData.cropW / 2, -cropData.cropH / 2, cropData.cropW, cropData.cropH
                    );
                    finalCtx.restore();
                    
                    // 生成blob并返回结果
                    const MAX_SIDE = 1200;
                    let exportCanvas = finalCanvas;
                    let exportWidth = cropData.outputWidth;
                    let exportHeight = cropData.outputHeight;
                    if (cropData.outputWidth > MAX_SIDE || cropData.outputHeight > MAX_SIDE) {
                        const scale = MAX_SIDE / Math.max(cropData.outputWidth, cropData.outputHeight);
                        exportWidth = Math.round(cropData.outputWidth * scale);
                        exportHeight = Math.round(cropData.outputHeight * scale);
                        exportCanvas = document.createElement('canvas');
                        exportCanvas.width = exportWidth;
                        exportCanvas.height = exportHeight;
                        exportCanvas.getContext('2d').drawImage(finalCanvas, 0, 0, exportWidth, exportHeight);
                    }
                    exportCanvas.toBlob((blob) => {
                        const objectUrl = URL.createObjectURL(blob);
                        uni.hideLoading();
                        this.$emit('success', {
                            url: objectUrl,
                            width: exportWidth,
                            height: exportHeight
                        });
                        this.$emit('input', false);
                    }, 'image/jpeg', this.quality);
                };
                img.onerror = () => {
                    uni.hideLoading();
                    uni.showToast({ title: '图片加载失败', icon: 'none' });
                    this.$emit('input', false);
                };
            }, 100);
            // #endif
        },
        cancel() {
            this.$emit('cancel', false)
            this.$emit('input', false)
        },
    }
};
</script>

<style lang="scss" scoped>
@import './index.scss';
</style>
