<script setup lang='ts'>
import { ref } from 'vue'
import {
  KeyboardState,
  MouseState,
  useAppStore,
  useDowningStore,
  useKeyboardStore,
  useMouseStore,
  useWinStore
} from '../store'
import {
  onHideRightMenuScroll,
  onShowRightMenu,
  RefreshScroll,
  RefreshScrollTo,
  TestCtrl,
  TestKey,
  TestKeyboardScroll,
  TestKeyboardSelect
} from '../utils/keyboardhelper'
import { Tooltip as AntdTooltip } from 'ant-design-vue'
import 'ant-design-vue/es/tooltip/style/css'
import { TestButton } from '../utils/mosehelper'

const viewlist = ref()
const inputsearch = ref()

const appStore = useAppStore()
const winStore = useWinStore()
const downingStore = useDowningStore()

const keyboardStore = useKeyboardStore()
keyboardStore.$subscribe((_m: any, state: KeyboardState) => {
  if (appStore.appTab != 'down' || appStore.GetAppTabMenu != 'DowningRight') return

  if (TestCtrl('a', state.KeyDownEvent, () => downingStore.mSelectAll())) return
  if (TestCtrl('f', state.KeyDownEvent, () => inputsearch.value.focus())) return
  if (TestKey('f3', state.KeyDownEvent, () => inputsearch.value.focus())) return
  if (TestKey(' ', state.KeyDownEvent, () => inputsearch.value.focus())) return

  if (TestKeyboardScroll(state.KeyDownEvent, viewlist.value, downingStore)) return
  if (TestKeyboardSelect(state.KeyDownEvent, viewlist.value, downingStore, null)) return
})

const rangIsSelecting = ref(false)
const rangSelectID = ref('')
const rangSelectStart = ref('')
const rangSelectEnd = ref('')
const rangSelectFiles = ref<{ [k: string]: any }>({})
const onSelectRangStart = () => {
  onHideRightMenuScroll()
  rangIsSelecting.value = !rangIsSelecting.value
  rangSelectID.value = ''
  rangSelectStart.value = ''
  rangSelectEnd.value = ''
  rangSelectFiles.value = {}
  downingStore.mRefreshListDataShow(false)
}
const onSelectCancel = () => {
  onHideRightMenuScroll()
  downingStore.ListSelected.clear()
  downingStore.ListFocusKey = ''
  downingStore.mRefreshListDataShow(false)
}

const onSelectRang = (file_id: string) => {
  if (rangIsSelecting.value && rangSelectID.value != '') {

    let startid = rangSelectID.value
    let endid = ''
    const s: { [k: string]: any } = {}
    const children = downingStore.ListDataShow
    let a = -1
    let b = -1
    for (let i = 0, maxi = children.length; i < maxi; i++) {
      if (children[i].DownID == file_id) a = i
      if (children[i].DownID == startid) b = i
      if (a > 0 && b > 0) break
    }
    if (a >= 0 && b >= 0) {
      if (a > b) {
        ;[a, b] = [b, a]
        endid = file_id
      } else {
        endid = startid
        startid = file_id
      }
      for (let n = a; n <= b; n++) {
        s[children[n].DownID] = true
      }
    }

    rangSelectStart.value = startid
    rangSelectEnd.value = endid
    rangSelectFiles.value = s
    downingStore.mRefreshListDataShow(false)
  }
}

const handleSelectAll = () => downingStore.mSelectAll()
const handleOrder = (order: string) => downingStore.mOrderListData(order)
const handleSelect = (file_id: string, event: any, isCtrl: boolean = false) => {
  onHideRightMenuScroll()
  if (rangIsSelecting.value) {
    if (!rangSelectID.value) {
      if (!downingStore.ListSelected.has(file_id)) downingStore.mMouseSelect(file_id, true, false)
      rangSelectID.value = file_id
      rangSelectStart.value = file_id
      rangSelectFiles.value = { [file_id]: true }
    } else {
      const start = rangSelectID.value
      const children = downingStore.ListDataShow
      let a = -1
      let b = -1
      for (let i = 0, maxi = children.length; i < maxi; i++) {
        if (children[i].DownID == file_id) a = i
        if (children[i].DownID == start) b = i
        if (a > 0 && b > 0) break
      }
      const fileList: string[] = []
      if (a >= 0 && b >= 0) {
        if (a > b) [a, b] = [b, a]
        for (let n = a; n <= b; n++) {
          fileList.push(children[n].DownID)
        }
      }
      downingStore.mRangSelect(file_id, fileList)
      rangIsSelecting.value = false
      rangSelectID.value = ''
      rangSelectStart.value = ''
      rangSelectEnd.value = ''
      rangSelectFiles.value = {}
    }
    downingStore.mRefreshListDataShow(false)
  } else {
    downingStore.mMouseSelect(file_id, event.ctrlKey || isCtrl, event.shiftKey)
    if (!downingStore.ListSelected.has(file_id)) downingStore.ListFocusKey = ''
  }
}

const mouseStore = useMouseStore()
mouseStore.$subscribe((_m: any, state: MouseState) => {
  if (appStore.appTab != 'down') return
  const mouseEvent = state.MouseEvent
  // console.log('MouseEvent', state.MouseEvent)
  if (TestButton(0, mouseEvent, () => {
    if (mouseEvent.srcElement) {
      // @ts-ignore
      if (mouseEvent.srcElement.className && mouseEvent.srcElement.className.toString().startsWith('arco-virtual-list')) {
        onSelectCancel()
      }
    }
  })) return
})

const handleStart = () => downingStore.mStartDowning()

const handleStartAll = () => downingStore.mStartAllDowning()

const handleStop = () => downingStore.mStopDowning()

const handleStopAll = async () => await downingStore.mStopAllDowning()

const handleDelete = async () => await downingStore.mDeleteDowning([...downingStore.ListSelected])

const handleDeleteAll = async () => await downingStore.mDeleteAllDowning()

const handleTop = () => downingStore.mOrderDowning([...downingStore.ListSelected])

const handleSearchInput = (value: string) => {
  downingStore.mSearchListData(value)
  RefreshScrollTo(viewlist.value.$el, 0)
}
const handleSearchEnter = (event: any) => {
  event.target.blur()
  RefreshScroll(viewlist.value.$el)
}
const handleRightClick = (e: { event: MouseEvent; node: any }) => {
  let key = e.node.key
  if (!downingStore.ListSelected.has(key)) downingStore.mMouseSelect(key, false, false)
  onShowRightMenu('downingrightmenu', e.event.clientX, e.event.clientY)
}
</script>

<template>
  <div style='height: 7px'></div>
  <div class='toppanbtns' style='height: 26px'>
    <div class='flex flexauto'></div>
    <div class='flex flexnoauto cellcount' title='总共文件数量'>
      <a-badge color='#637dff' :text="'总数 ' + downingStore.ListStats.count" />
    </div>
    <div class='flex flexnoauto cellcount' title='正在执行下载数'>
      <a-badge color='#637dff' :text="'下载 ' + downingStore.ListStats.runningCount" />
    </div>
    <div class='flex flexnoauto cellcount' title='总共文件大小'>
      <a-badge color='#637dff' :text="'大小 ' + downingStore.ListStats.totalSizeStr" />
    </div>
  </div>
  <div style='height: 14px'></div>
  <div class='toppanbtns' style='height: 26px'>
    <div class='toppanbtn' v-show='downingStore.IsListSelected'>
      <a-button type='text' size='small' tabindex='-1' @click='handleStart'><i class='iconfont iconstart' />开始
      </a-button>
      <a-button type='text' size='small' tabindex='-1' @click='handleStop'><i class='iconfont iconpause' />暂停
      </a-button>
      <a-button type='text' size='small' tabindex='-1' @click='handleTop'><i class='iconfont iconyouxian' />优先传输
      </a-button>
      <a-button type='text' size='small' tabindex='-1' @click='handleDelete'><i class='iconfont icondelete' />删除
      </a-button>
      <a-button type='text' size='small' tabindex='-1'><i class='iconfont icondian' /></a-button>

      <a-button type='text' size='small' tabindex='-1' @click='handleStartAll'><i class='iconfont iconstart' />开始全部
      </a-button>
      <a-button type='text' size='small' tabindex='-1' @click='handleStopAll'><i class='iconfont iconpause' />暂停全部
      </a-button>
      <a-button type='text' size='small' tabindex='-1' @click='handleDeleteAll'><i class='iconfont icondelete' />删除全部
      </a-button>
    </div>
    <div class='toppanbtn' v-show='!downingStore.IsListSelected'>
      <a-button type='text' size='small' tabindex='-1' @click='handleStartAll'><i class='iconfont iconstart' />开始全部
      </a-button>
      <a-button type='text' size='small' tabindex='-1' @click='handleStopAll'><i class='iconfont iconpause' />暂停全部
      </a-button>
      <a-button type='text' size='small' tabindex='-1' @click='handleDeleteAll'><i class='iconfont icondelete' />删除全部
      </a-button>
    </div>
    <div style='flex-grow: 1'></div>
    <div class='toppanbtn'>
      <a-input-search
        tabindex='-1'
        ref='inputsearch'
        size='small'
        title='Ctrl+F / F3 / Space'
        placeholder='快速筛选'
        :model-value='downingStore.ListSearchKey'
        @input='(val:any)=>handleSearchInput(val as string)'
        @press-enter='handleSearchEnter'
        @keydown.esc='($event.target as any).blur()'
      />
    </div>
    <div></div>
  </div>
  <div style='height: 9px'></div>
  <div class='toppanarea'>
    <div style='margin: 0 3px'>
      <AntdTooltip title='点击全选' placement='left'>
        <a-button shape='circle' type='text' tabindex='-1' class='select all' @click='handleSelectAll' title='Ctrl+A'>
          <i :class="downingStore.IsListSelectedAll ? 'iconfont iconrsuccess' : 'iconfont iconpic2'" />
        </a-button>
      </AntdTooltip>
    </div>
    <div class='selectInfo'>{{ downingStore.ListDataSelectCountInfo }}</div>
    <div style='margin: 0 2px'>
      <AntdTooltip placement='rightTop'>
        <a-button shape='square' type='text' tabindex='-1' class='qujian'
                  :status="rangIsSelecting ? 'danger' : 'normal'" title='Ctrl+Q' @click='onSelectRangStart'>
          {{ rangIsSelecting ? '取消选择' : '区间选择' }}
        </a-button>
        <template #title>
          <div>
            第1步: 点击 区间选择 这个按钮
            <br />
            第2步: 鼠标点击一个文件
            <br />
            第3步: 移动鼠标点击另外一个文件
          </div>
        </template>
      </AntdTooltip>
      <a-button shape='square' v-if='!rangIsSelecting && downingStore.ListSelected.size > 0' type='text' tabindex='-1' class='qujian'
                status='normal' @click='onSelectCancel'>
        取消已选
      </a-button>
    </div>
  </div>
  <div class='toppanlist' @keydown.space.prevent='() => true'>
    <a-list
      ref='viewlist'
      :bordered='false'
      :split='false'
      :max-height='winStore.GetListHeightNumber'
      :virtualListProps="{
        height: winStore.GetListHeightNumber,
        isStaticItemHeight: true,
        estimatedSize: 50,
        threshold: 1,
        itemKey: 'DownID'
      }"
      style='width: 100%'
      :data='downingStore.ListDataShow'
      :loading='downingStore.ListLoading'
      tabindex='-1'
      @scroll='onHideRightMenuScroll'
    >
      <template #item='{ item, index }'>
        <div :key='item.DownID' class='listitemdiv' :data-id='item.DownID'>
          <div
            :class="'fileitem' + (downingStore.ListSelected.has(item.DownID) ? ' selected' : '') + (downingStore.ListFocusKey == item.DownID ? ' focus' : '')"
            @click='handleSelect(item.DownID, $event)'
            @mouseover='onSelectRang(item.DownID)'
            @contextmenu='(event:MouseEvent)=>handleRightClick({event,node:{key:item.DownID}} )'
          >
            <div
              :class="'rangselect ' + (rangSelectFiles[item.DownID] ? (rangSelectStart == item.DownID ? 'rangstart' : rangSelectEnd == item.DownID ? 'rangend' : 'rang') : '')">
              <a-button shape='circle' type='text' tabindex='-1' class='select' :title='index'
                        @click.prevent.stop='handleSelect(item.DownID, $event, true)'>
                <i :class="downingStore.ListSelected.has(item.DownID) ? 'iconfont iconrsuccess' : 'iconfont iconpic2'" />
              </a-button>
            </div>
            <div class='fileicon'>
              <i :class="'iconfont ' + item.Info.icon" aria-hidden='true'></i>
            </div>
            <div class='filename'>
              <div :title='item.Info.localFilePath'>
                {{ item.Info.name }}
              </div>
            </div>
            <div class='cell filesize'>{{ item.Info.sizestr }}</div>
            <div class='downprogress'>
              <div class='transfering-state'>
                <p class='text-state'>{{ item.Down.DownState }}</p>
                <div class='progress-total'>
                  <div
                    :class="item.Down.IsDowning ? 'progress-current active' : item.Down.IsCompleted ? 'progress-current succeed' : item.Down.IsFailed ? 'progress-current error' : 'progress-current'"
                    :style="'width: ' + item.Down.DownProcess.toString() + '%'"
                  ></div>
                </div>
                <p class='text-error'>{{ item.Down.FailedMessage }}</p>
              </div>
            </div>
            <div class='downspeed'>{{ item.Down.DownSpeedStr }}</div>
          </div>
        </div>
      </template>
    </a-list>
    <a-dropdown id='downingrightmenu' class='rightmenu' :popup-visible='true' tabindex='-1' :draggable='false'
                style='z-index: -1; left: -200px; opacity: 0'>
      <template #content>
        <a-doption @click='handleStart'>
          <template #icon><i class='iconfont iconstart' /></template>
          <template #default>开始</template>
        </a-doption>
        <a-doption @click='handleStop'>
          <template #icon><i class='iconfont iconpause' /></template>
          <template #default>暂停</template>
        </a-doption>
        <a-doption @click='handleTop'>
          <template #icon><i class='iconfont iconyouxian' /></template>
          <template #default>优先传输</template>
        </a-doption>
        <a-doption @click='handleDelete' class='danger'>
          <template #icon><i class='iconfont icondelete' /></template>
          <template #default>删除</template>
        </a-doption>
      </template>
    </a-dropdown>
  </div>
</template>

<style>
.cellcount {
  align-items: center;
  margin-right: 16px;
}

.cellcount .arco-badge .arco-badge-status-text {
  margin-left: 4px;
  color: var(--color-text-3);
  line-height: 26px;
}

body[arco-theme='dark'] .toppanarea .cell {
  color: rgba(211, 216, 241, 0.45);
}

.cell {
  color: var(--color-text-3);
  overflow: hidden;
  text-align: center;
  flex-grow: 0;
  flex-shrink: 0;
  display: inline-block;
  line-height: 18px;
  min-height: 18px;
  padding: 0 4px;
  justify-content: center;
}

.cell.filesize {
  font-size: 16px;
  width: 86px;
  text-align: right;
  flex-shrink: 0;
  flex-grow: 0;
  margin-right: 16px;
}

.cell.tiquma {
  width: 60px;
  font-size: 12px;
}

.cell.count {
  width: 60px;
  font-size: 12px;
}

.cell.sharetime {
  width: 80px;
  font-size: 12px;
  line-height: 14px;
  text-align: right;
  word-wrap: break-word;
  word-break: keep-all;
}

.cell.sharetime.active {
  color: rgb(217, 48, 37);
}

.cell.sharestate {
  width: 70px;
  font-size: 12px;
}

.cell.sharestate.active {
  color: rgb(var(--primary-6));
}

.cell.sharestate.forbidden {
  color: rgb(217, 48, 37);
}

.cell.sharestate.deleted {
  text-decoration: line-through;
}

.cell.p5 {
  width: 5px;
}

.cell.pr {
  width: 12px;
}

.toppanarea .cell.order {
  cursor: pointer;
}

.toppanarea .cell.order:hover {
  color: rgb(var(--primary-6));
}

.downprogress {
  width: 90px;
  flex-shrink: 0;
  flex-grow: 0;
  margin-right: 8px;
}

.downspeed {
  width: 126px;
  font-size: 22px;
  color: var(--color-text-4);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: clip;
  flex-shrink: 0;
  flex-grow: 0;
  text-align: right;
}

.transfering-state {
  display: block;
  width: 100%;
  overflow: visible;
}

.text-state {
  font-size: 12px;
  line-height: 16px;
  color: var(--color-text-3);
  max-width: 100%;
  overflow: hidden;
  white-space: nowrap;
  -o-text-overflow: ellipsis;
  text-overflow: ellipsis;
  height: 16px;
  margin: 0;
}

.text-error {
  color: #f35b51;
  font-size: 12px;
  line-height: 16px;
  width: 100%;
  overflow: visible;
  white-space: nowrap;
  height: 16px;
  margin: 0;
}

.progress-total {
  width: 100%;
  height: 3px;
  background: var(--color-text-4);
  border-radius: 1.5px;
  margin-top: 2px;
  position: relative;
}

.progress-total .progress-current {
  position: absolute;
  top: 0;
  left: 0;
  max-width: 100%;
  min-width: 6px;
  height: 100%;
  border-radius: 1.5px;
  background: var(--color-primary-light-4);
  -webkit-transition: width 0.3s ease, opacity 0.3s ease;
  -o-transition: width 0.3s ease, opacity 0.3s ease;
  transition: width 0.3s ease, opacity 0.3s ease;
}

.progress-total .progress-current.succeed {
  background: #099970;
}

.progress-total .progress-current.error {
  background: #f35b51;
}

.progress-total .progress-current.active {
  background: linear-gradient(270deg, #ffba7a 0%, #ff74c7 8.56%, #637dff 26.04%, rgba(99, 125, 255, 0.2) 100%);
}

.downHideTip {
  text-align: center;
  padding: 8px;
  opacity: 0.5;
}

</style>
