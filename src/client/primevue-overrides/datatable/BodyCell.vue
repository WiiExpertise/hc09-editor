<template>
    <td :style="containerStyle" :class="containerClass" @click="onClick" @keydown="onKeyDown" role="cell">
        <span v-if="responsiveLayout === 'stack'" class="p-column-title">{{columnProp('header')}}</span>
        <component :is="column.children.body" :data="rowData" :column="column" :index="rowIndex" :frozenRow="frozenRow" v-if="column.children && column.children.body && !d_editing" />
        <component :is="column.children.editor" :data="rowData" :column="column" :index="rowIndex" :frozenRow="frozenRow" v-else-if="column.children && column.children.editor && d_editing" />
        <template v-else-if="columnProp('selectionMode')">
            <DTRadioButton :value="rowData" :checked="selected" @change="toggleRowWithRadio" v-if="column.props.selectionMode === 'single'" />
            <DTCheckbox :value="rowData" :checked="selected" @change="toggleRowWithCheckbox" v-else-if="column.props.selectionMode ==='multiple'" />
        </template>
        <template v-else-if="columnProp('rowReorder')">
            <i :class="['p-datatable-reorderablerow-handle', (columnProp('rowReorderIcon') || 'pi pi-bars')]"></i>
        </template>
        <template v-else-if="columnProp('expander')">
            <button class="p-row-toggler p-link" @click="toggleRow" type="button" v-ripple>
                <span :class="rowTogglerIcon"></span>
            </button>
        </template>
        <template v-else-if="editMode === 'row' && columnProp('rowEditor')">
            <button class="p-row-editor-init p-link" v-if="!d_editing" @click="onRowEditInit" type="button" v-ripple>
                <span class="p-row-editor-init-icon pi pi-fw pi-pencil"></span>
            </button>
            <button class="p-row-editor-save p-link" v-if="d_editing" @click="onRowEditSave" type="button" v-ripple>
                <span class="p-row-editor-save-icon pi pi-fw pi-check"></span>
            </button>
            <button class="p-row-editor-cancel p-link" v-if="d_editing" @click="onRowEditCancel" type="button" v-ripple>
                <span class="p-row-editor-cancel-icon pi pi-fw pi-times"></span>
            </button>
        </template>
        <template v-else>{{resolveFieldData()}}</template>
    </td>
</template>

<script>
import {DomHandler,ObjectUtils} from 'primevue/utils';
import OverlayEventBus from 'primevue/overlayeventbus';
import RowRadioButton from './RowRadioButton.vue';
import RowCheckbox from './RowCheckbox.vue';
import Ripple from 'primevue/ripple';

export default {
    name: 'BodyCell',
    emits: ['cell-edit-init', 'cell-edit-complete', 'cell-edit-cancel', 'row-edit-init', 'row-edit-save', 'row-edit-cancel', 'editing-cell-change',
            'row-toggle', 'radio-change', 'checkbox-change'],
    props: {
        rowData: {
            type: Object,
            default: null
        },
        column: {
            type: Object,
            default: null
        },
        frozenRow: {
            type: Boolean,
            default: false
        },
        rowIndex: {
            type: Number,
            default: null
        },
        index: {
            type: Number,
            default: null
        },
        rowTogglerIcon: {
            type: Array,
            default: null
        },
        selected: {
            type: Boolean,
            default: false
        },
        editing: {
            type: Boolean,
            default: false
        },
        editMode: {
            type: String,
            default: null
        },
        responsiveLayout: {
            type: String,
            default: 'stack'
        }
    },
    documentEditListener: null,
    selfClick: false,
    overlayEventListener: null,
    data() {
        return {
            d_editing: this.editing,
            styleObject: {}
        }
    },
    watch: {
        editing(newValue) {
            this.d_editing = newValue;
        }
    },
    mounted() {
        if (this.columnProp('frozen')) {
            this.updateStickyPosition();
        }
    },
    updated() {
        if (this.columnProp('frozen')) {
            this.updateStickyPosition();
        }
    },
    beforeUnmount() {
        if (this.overlayEventListener) {
            OverlayEventBus.off('overlay-click', this.overlayEventListener);
            this.overlayEventListener = null;
        }
    },
    methods: {
        columnProp(prop) {
            return this.column.props ? ((this.column.type.props[prop].type === Boolean && this.column.props[prop] === '') ? true : this.column.props[prop]) : null;
        },
        resolveFieldData() {
            return ObjectUtils.resolveFieldData(this.rowData, this.columnProp('field'));
        },
        toggleRow(event) {
            this.$emit('row-toggle', {
                originalEvent: event,
                data: this.rowData
            });
        },
        toggleRowWithRadio(event) {
            this.$emit('radio-change', event);
        },
        toggleRowWithCheckbox(event) {
            this.$emit('checkbox-change', event);
        },
        isEditable() {
            return this.column.children && this.column.children.editor != null;
        },
        bindDocumentEditListener() {
            if (!this.documentEditListener) {
                let drag = false;

                this.mouseMoveListener = () => {
                    console.log('drag');
                    drag = true;
                };

                this.mouseUpListener = () => {
                    if (!this.selfClick) {
                        this.completeEdit(event, 'outside');
                    }

                    this.selfClick = false;

                    document.removeEventListener('mousemove', this.mouseMoveListener);
                    document.removeEventListener('mouseup', this.mouseUpListener);
                };

                this.documentEditListener = (event) => {
                    document.addEventListener('mousemove', this.mouseMoveListener);
                    document.addEventListener('mouseup', this.mouseUpListener);
                };

                document.addEventListener('mousedown', this.documentEditListener);
            }
        },
        unbindDocumentEditListener() {
            if (this.documentEditListener) {
                document.removeEventListener('mousedown', this.documentEditListener);
                this.documentEditListener = null;
                this.selfClick = false;
            }
        },
        switchCellToViewMode() {
            this.d_editing = false;
            this.unbindDocumentEditListener();
            this.$emit('editing-cell-change', {rowIndex: this.rowIndex, cellIndex: this.index, editing: false});
            OverlayEventBus.off('overlay-click', this.overlayEventListener);
            this.overlayEventListener = null;
        },
        onClick(event) {
            if (this.editMode === 'cell' && this.isEditable()) {
                this.selfClick = true;

                if (!this.d_editing) {
                    this.d_editing = true;
                    this.bindDocumentEditListener();
                    this.$emit('cell-edit-init', {originalEvent: event, data: this.rowData, field: this.columnProp('field'), index: this.rowIndex});
                    this.$emit('editing-cell-change', {rowIndex: this.rowIndex, cellIndex: this.index, editing: true});

                    this.overlayEventListener = (e) => {
                        if (this.$el && this.$el.contains(e.target)) {
                            this.selfClick = true;
                        }
                    }
                    OverlayEventBus.on('overlay-click', this.overlayEventListener);
                }
            }
        },
        completeEdit(event, type) {
            let completeEvent = {
                originalEvent: event,
                data: this.rowData,
                field: this.columnProp('field'),
                index: this.rowIndex,
                type: type,
                defaultPrevented: false,
                preventDefault: function() {
                    this.defaultPrevented = true;
                }
            };

            this.$emit('cell-edit-complete', completeEvent);

            if (!completeEvent.defaultPrevented) {
                this.switchCellToViewMode();
            }
        },
        onKeyDown(event) {
            if (this.editMode === 'cell') {
                switch (event.which) {
                    case 13:
                        this.completeEdit(event, 'enter');
                    break;

                    case 27:
                        this.switchCellToViewMode();
                        this.$emit('cell-edit-cancel', {originalEvent: event, data: this.rowData, field: this.columnProp('field'), index: this.rowIndex});
                    break;

                    case 9:
                        this.completeEdit(event, 'tab');

                        if (event.shiftKey)
                            this.moveToPreviousCell(event);
                        else
                            this.moveToNextCell(event);
                    break;
                }
            }
        },
        moveToPreviousCell(event) {
            let currentCell = this.findCell(event.target);
            let targetCell = this.findPreviousEditableColumn(currentCell);

            if (targetCell) {
                DomHandler.invokeElementMethod(targetCell, 'click');
                event.preventDefault();
            }
        },
        moveToNextCell(event) {
            let currentCell = this.findCell(event.target);
            let targetCell = this.findNextEditableColumn(currentCell);

            if (targetCell) {
                DomHandler.invokeElementMethod(targetCell, 'click');
                event.preventDefault();
            }
        },
        findCell(element) {
            if (element) {
                let cell = element;
                while (cell && !DomHandler.hasClass(cell, 'p-cell-editing')) {
                    cell = cell.parentElement;
                }

                return cell;
            }
            else {
                return null;
            }
        },
        findPreviousEditableColumn(cell) {
            let prevCell = cell.previousElementSibling;

            if (!prevCell) {
                let previousRow = cell.parentElement.previousElementSibling;
                if (previousRow) {
                    prevCell = previousRow.lastElementChild;
                }
            }

            if (prevCell) {
                if (DomHandler.hasClass(prevCell, 'p-editable-column'))
                    return prevCell;
                else
                    return this.findPreviousEditableColumn(prevCell);
            }
            else {
                return null;
            }
        },
        findNextEditableColumn(cell) {
            let nextCell = cell.nextElementSibling;

            if (!nextCell) {
                let nextRow = cell.parentElement.nextElementSibling;
                if (nextRow) {
                    nextCell = nextRow.firstElementChild;
                }
            }

            if (nextCell) {
                if (DomHandler.hasClass(nextCell, 'p-editable-column'))
                    return nextCell;
                else
                    return this.findNextEditableColumn(nextCell);
            }
            else {
                return null;
            }
        },
        isEditingCellValid() {
            return (DomHandler.find(this.$el, '.p-invalid').length === 0);
        },
        onRowEditInit(event) {
            this.$emit('row-edit-init', {originalEvent: event, data: this.rowData, field: this.columnProp('field'), index: this.rowIndex});
        },
        onRowEditSave(event) {
            this.$emit('row-edit-save', {originalEvent: event, data: this.rowData, field: this.columnProp('field'), index: this.rowIndex});
        },
        onRowEditCancel(event) {
            this.$emit('row-edit-cancel', {originalEvent: event, data: this.rowData, field: this.columnProp('field'), index: this.rowIndex});
        },
        updateStickyPosition() {
            if (this.columnProp('frozen')) {
                let align = this.columnProp('alignFrozen');
                if (align === 'right') {
                    let right = 0;
                    let next = this.$el.nextElementSibling;
                    if (next) {
                        right = DomHandler.getOuterWidth(next) + parseFloat(next.style.left);
                    }
                    this.styleObject.right = right + 'px';
                }
                else {
                    let left = 0;
                    let prev = this.$el.previousElementSibling;
                    if (prev) {
                        left = DomHandler.getOuterWidth(prev) + parseFloat(prev.style.left);
                    }
                    this.styleObject.left = left + 'px';
                }
            }
        }
    },
    computed: {
        containerClass() {
            return [this.columnProp('bodyClass'), this.columnProp('class'), {
                'p-selection-column': this.columnProp('selectionMode') != null,
                'p-editable-column': this.isEditable(),
                'p-cell-editing': this.d_editing,
                'p-frozen-column': this.columnProp('frozen')
            }];
        },
        containerStyle() {
            let bodyStyle = this.columnProp('bodyStyle');
            let columnStyle = this.columnProp('style');

            return this.columnProp('frozen') ? [columnStyle, bodyStyle, this.styleObject]: [columnStyle, bodyStyle];
        }
    },
    components: {
        'DTRadioButton': RowRadioButton,
        'DTCheckbox': RowCheckbox
    },
    directives: {
        'ripple': Ripple
    }
}
</script>
