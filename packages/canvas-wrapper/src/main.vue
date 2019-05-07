<template>
  <!--画布区域-->
  <div id="canvasHelpWrapper" class="canvasHelpWrapper">
    <div class="canvas-wrapper canvasSection" id="canvasSection"
         v-droppable="{onDrop:'dropCallback',onOver: 'overCallback', onOut: 'outCallback'}"
         data-model="droppedElement"
         data-drop="true"
         @scroll.passive="fnScroll">
      <div class="canvas-message" id="model-modified-date"></div>

      <div id="flow_op_btns" v-show="!btn_visibile.hide_shape_buttons">
        <!--删除按钮-->
        <div class="Oryx_button" id="delete-button"
             :title="t('BUTTON.ACTION.DELETE.TOOLTIP')"
             @click="deleteShape">
          <img src="@/assets/images/delete.png"/>
        </div>
        <!--设置修改形状-->
        <div v-if="!btn_visibile.hide_morph_buttons"
             class="Oryx_button" id="morph-button"
             :title="t('BUTTON.ACTION.MORPH.TOOLTIP')"
             @click="morphShape">

          <el-dropdown trigger="click" @command="handleCommand">
      <span class="el-dropdown-link">
        <img src="@/assets/images/wrench.png"/>
        <!--下拉菜单<i class="el-icon-arrow-down el-icon&#45;&#45;right"></i>-->
      </span>
            <el-dropdown-menu slot="dropdown">
              <el-dropdown-item v-for="item in morphShapes" :key="item.id" :command="item">
                <img width="16px;" height="16px;"
                     :src="require(`@/assets/images/bpmn2.0/icons/${item.icon}`)"/>
                {{item.name}}
              </el-dropdown-item>
            </el-dropdown-menu>
          </el-dropdown>

        </div>
        <!--编辑-->
        <div v-if="!btn_visibile.hide_edit_buttons"
             class="Oryx_button" id="edit-button" @click="editShape">
          <img src="@/assets/images/pencil.png"/>
        </div>
      </div>

      <!--v-draggable="{onStart:'startDragCallbackQuickMenu', onDrag:'dragCallbackQuickMenu',-->
      <!--revert: 'invalid', helper: 'clone', opacity : 0.5}"-->
      <div id="flow_add_btns" v-show="!(btn_visibile.hide_shape_buttons || btn_visibile.hide_flow_add_btns)">
        <div v-for="item in quickMenuItems"
             class="Oryx_button"
             :key="item.id"
             :id="item.id"
             :title="item.description"
             @click="quickAddItem(item.id)"
             data-model="draggedElement"
             data-drag="true"
             v-draggable="{onStart:'startDragCallbackQuickMenu', onDrag:'dragCallbackQuickMenu',
           revert: 'invalid', helper: 'clone', opacity : 0.5}">
          <img :src="require(`@/assets/images/bpmn2.0/icons/${item.icon}`)"/>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
  import ORYX_CONFIG from 'src/oryx/CONFIG'
  import FLOW_eventBus from 'src/flowable/FLOW_eventBus'
  import { EventBus } from 'src/EventBus'
  import { _debounce, getAdditionalIEZoom } from 'src/Util'
  import Locale from 'src/mixins/locale'
  import { draggable, droppable } from 'src/directives/drag-drop'

  export default {
    name: 'canvasWrapper',
    data () {
      return {
        morphShapes: [],
        currentSelectedMorph: null,
        newShape: null,
        btn_visibile: {
          hide_shape_buttons: true,
          hide_flow_add_btns: true,
          hide_morph_buttons: true,
          hide_edit_buttons: true
        },
        dragCanContain: false,
        dragModeOver: false,
        quickMenu: undefined
      }
    },
    mixins: [Locale],
    directives: {
      draggable,
      droppable
    },
    props: {
      editorManager: {}
    },
    mounted () {
      // 隐藏画布节点的快捷按钮
      FLOW_eventBus.addListener(ORYX_CONFIG.EVENT_TYPE_HIDE_SHAPE_BUTTONS, (btns) => {
        btns.map(btn => {
          this.btn_visibile[btn.type] = btn.status
        })
      })

      EventBus.$on('UPDATE_dragModeOver', (status) => {
        this.dragModeOver = status
        EventBus.$emit('UPDATE_dragModeOver_forDragItem', status)
      })
      EventBus.$on('UPDATE_dragCanContain', (status) => {
        this.dragCanContain = status
      })
      EventBus.$on('UPDATE_quickMenu', (data) => {
        this.quickMenu = data
      })
    },
    computed: {
      modelData () {
        return this.editorManager ? this.editorManager.getBaseModelData() : []
      },
      quickMenuItems () {
        return this.editorManager ? this.editorManager.quickMenuItems : []
      }
    },
    methods: {
      editShape () {
        this.editorManager.edit(this.selectedShape.resourceId)
      },
      fnScroll () {
        // Hides the resizer and quick menu items during scrolling
        const selectedElements = this.editorManager.getSelection()
        const subSelectionElements = this.editorManager.getSubSelection()

        this.selectedElements = selectedElements
        this.subSelectionElements = subSelectionElements
        if (selectedElements && selectedElements.length > 0) {
          this.selectedElementBeforeScrolling = selectedElements[0]
        }
        this.btn_visibile.hide_shape_buttons = true

        jQuery('.resizer_southeast').each(function (i, obj) {
          this.orginalResizerSEStyle = obj.style.display
          obj.style.display = 'none'
        })
        jQuery('.resizer_northwest').each(function (i, obj) {
          this.orginalResizerNWStyle = obj.style.display
          obj.style.display = 'none'
        })
        jQuery('.resizer_south').each(function (i, obj) {
          this.orginalResizerNWStyle = obj.style.display
          obj.style.display = 'none'
        })
        jQuery('.resizer_north').each(function (i, obj) {
          this.orginalResizerNWStyle = obj.style.display
          obj.style.display = 'none'
        })
        this.editorManager.handleEvents({ type: ORYX_CONFIG.EVENT_CANVAS_SCROLL })
        this.fnHandleScrollDebounce()
      },
      fnHandleScrollDebounce: _debounce(function (_type, index, item) {
        // Puts the quick menu items and resizer back when scroll is stopped.
        // this.editorManager.setSelection([])
        // needed cause it checks for element changes and does nothing if the elements are the same
        // this.editorManager.setSelection(this.selectedElements, this.subSelectionElements)
        this.editorManager.updateOryxButtonPosition(this.selectedElements)
        this.selectedElements = undefined
        this.subSelectionElements = undefined

        function handleDisplayProperty (obj) {
          if (jQuery(obj).position().top > 0) {
            obj.style.display = 'block'
          } else {
            obj.style.display = 'none'
          }
        }

        // jQuery('.Oryx_button').each(function(i, obj) {
        //   handleDisplayProperty(obj);
        // });
        jQuery('.resizer_southeast').each(function (i, obj) {
          handleDisplayProperty(obj)
        })
        jQuery('.resizer_northwest').each(function (i, obj) {
          handleDisplayProperty(obj)
        })
        jQuery('.resizer_south').each(function (i, obj) {
          handleDisplayProperty(obj)
        })
        jQuery('.resizer_north').each(function (i, obj) {
          handleDisplayProperty(obj)
        })
      }, 200),
      deleteShape () {
        this.editorManager.flowToolbarEvent({ '$scope': this, 'editorManager': this.editorManager })
      },
      morphShape () {
        var shapes = this.editorManager.getSelection()
        if (shapes && shapes.length == 1) {
          this.currentSelectedShape = shapes.first()
          let currentSelectedShapeId = this.currentSelectedShape.getStencil().idWithoutNs()
          var stencilItem = this.editorManager.getStencilItemById(currentSelectedShapeId)

          var morphShapes = []
          const morphRoles = this.editorManager.morphRoles
          // && morphRoles[i].id !== currentSelectedShapeId
          for (let i = 0; i < morphRoles.length; i++) {
            if (morphRoles[i].role === stencilItem.morphRole) {
              let ary = morphRoles[i].morphOptions.slice()
              for (let y = 0; y < ary.length; y++) {
                if (ary[y].id != currentSelectedShapeId) {
                  morphShapes.push(ary[y])
                }
              }
            }
          }
          this.morphShapes = morphShapes
        }
      },
      // 切换元素类型
      handleCommand (item) {
        let stencil = undefined
        const stencilSets = this.editorManager.getStencilSets().values()

        const stencilId = item.genericTaskId || item.id

        for (let i = 0; i < stencilSets.length; i++) {
          let stencilSet = stencilSets[i]
          let nodes = stencilSet.nodes()
          for (let j = 0; j < nodes.length; j++) {
            if (nodes[j].idWithoutNs() === stencilId) {
              stencil = nodes[j]
              break
            }
          }
        }

        if (!stencil) return

        // Create and execute command (for undo/redo)
        // const command = new MorphTo(this.currentSelectedShape, stencil, this.editorManager.getEditor())
        // this.editorManager.executeCommands([command])
        this.editorManager.assignCommand('MorphTo', this.currentSelectedShape, stencil, this.editorManager.getEditor())

      },
      dropCallback (event, ui) {
        this.editorManager.handleEvents({
          type: ORYX_CONFIG.EVENT_HIGHLIGHT_HIDE,
          highlightId: 'shapeRepo.attached'
        })
        this.editorManager.handleEvents({
          type: ORYX_CONFIG.EVENT_HIGHLIGHT_HIDE,
          highlightId: 'shapeRepo.added'
        })
        this.editorManager.handleEvents({
          type: ORYX_CONFIG.EVENT_HIGHLIGHT_HIDE,
          highlightId: 'shapeMenu'
        })

        this.editorManager.dispatchFlowEvent(ORYX_CONFIG.EVENT_TYPE_HIDE_SHAPE_BUTTONS, [
          { type: 'hide_shape_buttons', status: true },
          { type: 'hide_flow_add_btns', status: true },
          { type: 'hide_morph_buttons', status: true },
          { type: 'hide_edit_buttons', status: true }
        ])

        // console.log('dragCanContain', this.dragCanContain)
        if (this.dragCanContain) {
          let item = this.editorManager.getStencilItemById(ui.draggable[0].id)
          let pos = { x: event.pageX, y: event.pageY }

          let additionalIEZoom = getAdditionalIEZoom()
          let screenCTM = this.editorManager.getCanvas().node.getScreenCTM()

          // Correcting the UpperLeft-Offset
          pos.x -= (screenCTM.e / additionalIEZoom)
          pos.y -= (screenCTM.f / additionalIEZoom)
          // Correcting the Zoom-Factor
          pos.x /= screenCTM.a
          pos.y /= screenCTM.d
          // Correcting the ScrollOffset
          pos.x -= document.documentElement.scrollLeft
          pos.y -= document.documentElement.scrollTop

          let parentAbs = this.editorManager.dragCurrentParent.absoluteXY()
          pos.x -= parentAbs.x
          pos.y -= parentAbs.y

          let containedStencil = undefined
          let stencilSets = this.editorManager.getStencilSets().values()
          for (let i = 0; i < stencilSets.length; i++) {
            let stencilSet = stencilSets[i]
            let nodes = stencilSet.nodes()
            for (let j = 0; j < nodes.length; j++) {
              if (nodes[j].idWithoutNs() === ui.draggable[0].id) {
                containedStencil = nodes[j]
                break
              }
            }

            if (!containedStencil) {
              let edges = stencilSet.edges()
              for (let j = 0; j < edges.length; j++) {
                if (edges[j].idWithoutNs() === ui.draggable[0].id) {
                  containedStencil = edges[j]
                  break
                }
              }
            }
          }

          if (!containedStencil) return

          if (this.quickMenu) {
            // 当拖拽的是快捷元素
            let shapes = this.editorManager.getSelection()
            if (shapes && shapes.length == 1) {
              let currentSelectedShape = shapes.first()
              let option = {}
              option.type = currentSelectedShape.getStencil().namespace() + ui.draggable[0].id
              option.namespace = currentSelectedShape.getStencil().namespace()
              option.connectedShape = currentSelectedShape
              option.parent = this.editorManager.dragCurrentParent
              option.containedStencil = containedStencil

              // If the ctrl key is not pressed,
              // snapp the new shape to the center
              // if it is near to the center of the other shape
              if (!event.ctrlKey) {
                // Get the center of the shape
                let cShape = currentSelectedShape.bounds.center()
                // Snapp +-20 Pixel horizontal to the center
                if (20 > Math.abs(cShape.x - pos.x)) {
                  pos.x = cShape.x
                }
                // Snapp +-20 Pixel vertical to the center
                if (20 > Math.abs(cShape.y - pos.y)) {
                  pos.y = cShape.y
                }
              }

              option.position = pos

              if (containedStencil.idWithoutNs() !== 'SequenceFlow' && containedStencil.idWithoutNs() !== 'Association' &&
                containedStencil.idWithoutNs() !== 'MessageFlow' && containedStencil.idWithoutNs() !== 'DataAssociation') {

                let args = { sourceShape: currentSelectedShape, targetStencil: containedStencil }
                let targetStencil = this.editorManager.getRules().connectMorph(args)
                if (!targetStencil) { // Check if there can be a target shape
                  return
                }
                option.connectingType = targetStencil.id()
              }

              // let command = new FLOWABLE_OPTIONS.CreateCommand(option, this.editorManager.dropTargetElement, pos, this.editorManager.getEditor())
              //
              // this.editorManager.executeCommands([command])
              this.editorManager.assignCommand('CreateCommand', option, this.editorManager.dropTargetElement, pos, this.editorManager.getEditor())
            }

          } else {
            let canAttach = false
            if (containedStencil.idWithoutNs() === 'BoundaryErrorEvent' || containedStencil.idWithoutNs() === 'BoundaryTimerEvent' ||
              containedStencil.idWithoutNs() === 'BoundarySignalEvent' || containedStencil.idWithoutNs() === 'BoundaryMessageEvent' ||
              containedStencil.idWithoutNs() === 'BoundaryCancelEvent' || containedStencil.idWithoutNs() === 'BoundaryCompensationEvent') {

              // Modify position, otherwise boundary event will get position related to left corner of the canvas instead of the container
              pos = this.editorManager.eventCoordinates(event)
              canAttach = true
            }

            let option = {}
            option['type'] = this.modelData.model.stencilset.namespace + item.id
            option['namespace'] = this.modelData.model.stencilset.namespace
            option['position'] = pos
            option['parent'] = this.editorManager.dragCurrentParent


            // Update canvas
            // let command = new commandClass(option, this.editorManager.dragCurrentParent, canAttach, pos, this.editorManager.getEditor())
            // this.editorManager.executeCommands([command])
            this.editorManager.assignCommand('CommandClass', option, this.editorManager.dragCurrentParent, canAttach, pos, this.editorManager.getEditor())

            // Fire event to all who want to know about this
            let dropEvent = {
              type: ORYX_CONFIG.EVENT_TYPE_ITEM_DROPPED,
              droppedItem: item,
              position: pos
            }
            this.editorManager.dispatchFlowEvent(dropEvent.type, dropEvent)
          }
        }
        this.editorManager.dragCurrentParent = undefined
        this.editorManager.dragCurrentParentId = undefined
        this.editorManager.dragCurrentParentStencil = undefined
        EventBus.$emit('UPDATE_dragCanContain', undefined)
        EventBus.$emit('UPDATE_quickMenu', undefined)
        this.editorManager.dropTargetElement = undefined
      },
      overCallback (event, ui) {
        EventBus.$emit('UPDATE_dragModeOver', true)
      },
      outCallback (event, ui) {
        EventBus.$emit('UPDATE_dragModeOver', false)
        console.log('out==============')
      },
      startDragCallbackQuickMenu (event, ui) {
        EventBus.$emit('UPDATE_dragModeOver', false)
        EventBus.$emit('UPDATE_quickMenu', true)
      },
      dragCallbackQuickMenu (event, ui) {
        console.log('dragCallbackQuickMenu==============')
        if (this.dragModeOver != false) {
          var coord = this.editorManager.eventCoordinatesXY(event.pageX, event.pageY)

          var additionalIEZoom = 1
          if (!isNaN(screen.logicalXDPI) && !isNaN(screen.systemXDPI)) {
            var ua = navigator.userAgent
            if (ua.indexOf('MSIE') >= 0) {
              // IE 10 and below
              var zoom = Math.round((screen.deviceXDPI / screen.logicalXDPI) * 100)
              if (zoom !== 100) {
                additionalIEZoom = zoom / 100
              }
            }
          }

          if (additionalIEZoom !== 1) {
            coord.x = coord.x / additionalIEZoom
            coord.y = coord.y / additionalIEZoom
          }

          var aShapes = this.editorManager.getCanvas().getAbstractShapesAtPosition(coord)

          if (aShapes.length <= 0) {
            if (event.helper) {
              EventBus.$emit('UPDATE_dragCanContain', false)
              return false
            }
          }

          if (this.editorManager.instanceofCanvas(aShapes[0])) {
            this.editorManager.getCanvas().setHightlightStateBasedOnX(coord.x)
          }

          let stencil = undefined
          let stencilSets = this.editorManager.getStencilSets().values()
          for (let i = 0; i < stencilSets.length; i++) {
            let stencilSet = stencilSets[i]
            let nodes = stencilSet.nodes()
            for (let j = 0; j < nodes.length; j++) {
              if (nodes[j].idWithoutNs() === event.target.id) {
                stencil = nodes[j]
                break
              }
            }

            if (!stencil) {
              var edges = stencilSet.edges()
              for (var j = 0; j < edges.length; j++) {
                if (edges[j].idWithoutNs() === event.target.id) {
                  stencil = edges[j]
                  break
                }
              }
            }
          }

          let candidate = aShapes.last()

          let isValid = false
          if (stencil.type() === 'node') {
            // check containment rules
            let canContain = this.editorManager.getRules().canContain({
              containingShape: candidate,
              containedStencil: stencil
            })

            let parentCandidate = aShapes.reverse().find(function (candidate) {
              return (this.editorManager.instanceofCanvas(candidate)
                || this.editorManager.instanceofNode(candidate)
                || this.editorManager.instanceofEdge(candidate))
            })

            if (!parentCandidate) {
              EventBus.$emit('UPDATE_dragCanContain', false)
              return false
            }

            this.editorManager.dragCurrentParent = parentCandidate
            this.editorManager.dragCurrentParentId = parentCandidate.id
            this.editorManager.dragCurrentParentStencil = parentCandidate.getStencil().id()
            EventBus.$emit('UPDATE_dragCanContain', canContain)
            this.editorManager.dropTargetElement = parentCandidate
            isValid = canContain

          } else { //Edge
            let shapes = this.editorManager.getSelection()
            if (shapes && shapes.length === 1) {
              let currentSelectedShape = shapes.first()
              let curCan = candidate
              let canConnect = false

              let targetStencil = this.editorManager.getStencilItemById(curCan.getStencil().idWithoutNs())
              if (targetStencil) {
                let associationConnect = false
                if (stencil.idWithoutNs() === 'Association' && (curCan.getStencil().idWithoutNs() === 'TextAnnotation' || curCan.getStencil().idWithoutNs() === 'BoundaryCompensationEvent')) {
                  associationConnect = true
                } else if (stencil.idWithoutNs() === 'DataAssociation' && curCan.getStencil().idWithoutNs() === 'DataStore') {
                  associationConnect = true
                }

                if (targetStencil.canConnectTo || associationConnect) {
                  while (!canConnect && curCan && !this.editorManager.instanceofCanvas(curCan)) {
                    candidate = curCan
                    //check connection rules
                    canConnect = this.editorManager.getRules().canConnect({
                      sourceShape: currentSelectedShape,
                      edgeStencil: stencil,
                      targetShape: curCan
                    })
                    curCan = curCan.parent
                  }
                }
              }
              let parentCandidate = this.editorManager.getCanvas()

              isValid = canConnect
              this.editorManager.dragCurrentParent = parentCandidate
              this.editorManager.dragCurrentParentId = parentCandidate.id
              this.editorManager.dragCurrentParentStencil = parentCandidate.getStencil().id()
              EventBus.$emit('canContain', canConnect)
              this.editorManager.dropTargetElement = candidate
            }

          }

          this.editorManager.handleEvents({
            type: ORYX_CONFIG.EVENT_HIGHLIGHT_SHOW,
            highlightId: 'shapeMenu',
            elements: [candidate],
            color: isValid ? ORYX_CONFIG.SELECTION_VALID_COLOR : ORYX_CONFIG.SELECTION_INVALID_COLOR
          })
        }
      },
      quickAddItem (newItemId) {
        console.log('Oryx_button:', newItemId)
        let shapes = this.editorManager.getSelection()
        if (shapes && shapes.length === 1) {
          this.currentSelectedShape = shapes.first()

          let containedStencil = undefined
          let stencilSets = this.editorManager.getStencilSets().values()
          for (let i = 0; i < stencilSets.length; i++) {
            let stencilSet = stencilSets[i]
            let nodes = stencilSet.nodes()
            for (let j = 0; j < nodes.length; j++) {
              if (nodes[j].idWithoutNs() === newItemId) {
                containedStencil = nodes[j]
                break
              }
            }
          }

          if (!containedStencil) return

          let option = {
            type: this.currentSelectedShape.getStencil().namespace() + newItemId,
            namespace: this.currentSelectedShape.getStencil().namespace()
          }
          option['connectedShape'] = this.currentSelectedShape
          option['parent'] = this.currentSelectedShape.parent
          option['containedStencil'] = containedStencil

          let args = { sourceShape: this.currentSelectedShape, targetStencil: containedStencil }
          let targetStencil = this.editorManager.getRules().connectMorph(args)

          // Check if there can be a target shape
          if (!targetStencil) {
            return
          }

          option['connectingType'] = targetStencil.id()
          this.editorManager.assignCommand('CreateCommand', option, undefined, undefined, this.editorManager.getEditor())
        }
      }
    }
  }
</script>

<style scoped>

</style>