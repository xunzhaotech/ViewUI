<!--
<template>
    <table cellspacing="0" cellpadding="0" border="0" :style="styleObject">
        <colgroup>
            <col v-for="(column, index) in columns" :width="setCellWidth(column)">
        </colgroup>
        <tbody :class="[prefixCls + '-tbody']">
            <template v-for="(row, index) in data">
                <table-tr
                    :draggable="draggable"
                    :row="row"
                    :key="rowKey ? row._rowKey : index"
                    :prefix-cls="prefixCls"
                    @mouseenter.native.stop="handleMouseIn(row._index)"
                    @mouseleave.native.stop="handleMouseOut(row._index)"
                    @click.native="clickCurrentRow(row._index)"
                    @dblclick.native.stop="dblclickCurrentRow(row._index)">
                    <td v-for="(column, colIndex) in columns" :class="alignCls(column, row)" v-bind="getSpan(row, column, index, colIndex)" v-if="showWithSpan(row, column, index, colIndex)">
                        <table-cell
                            :fixed="fixed"
                            :prefix-cls="prefixCls"
                            :row="row"
                            :key="column._columnKey"
                            :column="column"
                            :natural-index="index"
                            :index="row._index"
                            :checked="rowChecked(row._index)"
                            :disabled="rowDisabled(row._index)"
                            :expanded="rowExpanded(row._index)"
                        ></table-cell>
                    </td>
                </table-tr>
                <tr v-if="rowExpanded(row._index)" :class="{[prefixCls + '-expanded-hidden']: fixed}">
                    <td :colspan="columns.length" :class="prefixCls + '-expanded-cell'">
                        <Expand :key="rowKey ? row._rowKey : index" :row="row" :render="expandRender" :index="row._index"></Expand>
                    </td>
                </tr>
            </template>
        </tbody>
    </table>
</template>
-->
<script>
    // todo :key="row"
    import TableTr from './table-tr.vue';
    import TableCell from './cell.vue';
    import Expand from './expand.js';
    import Mixin from './mixin';

    export default {
        name: 'TableBody',
        mixins: [ Mixin ],
        components: { TableCell, Expand, TableTr },
        props: {
            prefixCls: String,
            styleObject: Object,
            columns: Array,
            data: Array,    // rebuildData
            objData: Object,
            columnsWidth: Object,
            fixed: {
                type: [Boolean, String],
                default: false
            },
            draggable: {
                type: Boolean,
                default: false
            },
            rowKey: {
                type: [Boolean, String],
                default: false
            }
        },
        computed: {
            expandRender () {
                let render = function () {
                    return '';
                };
                for (let i = 0; i < this.columns.length; i++) {
                    const column = this.columns[i];
                    if (column.type && column.type === 'expand') {
                        if (column.render) render = column.render;
                    }
                }
                return render;
            }
        },
        methods: {
            rowChecked (_index) {
                return this.objData[_index] && this.objData[_index]._isChecked;
            },
            rowDisabled (_index) {
                return this.objData[_index] && this.objData[_index]._isDisabled;
            },
            rowExpanded (_index) {
                return this.objData[_index] && this.objData[_index]._isExpanded;
            },
            handleMouseIn (_index, event) {
                event.stopPropagation();
                this.$parent.handleMouseIn(_index);
            },
            handleMouseOut (_index, event) {
                event.stopPropagation();
                this.$parent.handleMouseOut(_index);
            },
            clickCurrentRow (_index) {
                this.$parent.clickCurrentRow(_index);
            },
            dblclickCurrentRow (_index, event) {
                event.stopPropagation();
                this.$parent.dblclickCurrentRow(_index);
            },
            getSpan (row, column, rowIndex, columnIndex) {
                const fn = this.$parent.spanMethod;
                if (typeof fn === 'function') {
                    const result = fn({
                        row,
                        column,
                        rowIndex,
                        columnIndex
                    });
                    let rowspan = 1;
                    let colspan = 1;
                    if (Array.isArray(result)) {
                        rowspan = result[0];
                        colspan = result[1];
                    } else if (typeof result === 'object') {
                        rowspan = result.rowspan;
                        colspan = result.colspan;
                    }
                    return {
                        rowspan,
                        colspan
                    };
                } else {
                    return {};
                }
            },
            showWithSpan (row, column, rowIndex, columnIndex) {
                const result = this.getSpan(row, column, rowIndex, columnIndex);
                return !(('rowspan' in result && result.rowspan === 0) || ('colspan' in result && result.colspan === 0));
            },
            isTrShow (rowKey) {
                let status = true;
                for (let i in this.objData) {
                    const row = this.objData[i];
                    const showChildren = row._isShowChildren;
                    if (row._rowKey === rowKey) {
                        status = status && showChildren;
                    } else if (row.children && row.children.length) {
                        status = this.getTrStatus(rowKey, row, status && showChildren);
                    }
                }
                return status;
            },
            getTrStatus (rowKey, data, parentStatus) {
                let status = parentStatus;
                if (data.children && data.children.length) {
                    data.children.forEach(row => {
                        const showChildren = row._isShowChildren;
                        if (row._rowKey === rowKey) {
                            status = status && showChildren;
                        } else if (row.children && row.children.length) {
                            status = this.getTrStatus(rowKey, row, status && showChildren);
                        }
                    });
                }
                return status;
            },
            getChildNode (h, data, nodes, level = 1) {
                if (data.children && data.children.length) {
                    data.children.forEach((row, index) => {
                        let $tds = [];

                        this.columns.forEach((column, colIndex) => {
                            if (this.showWithSpan(row, column, index, colIndex)) {
                                const $tableCell = h(TableCell, {
                                    props: {
                                        fixed: this.fixed,
                                        'prefix-cls': this.prefixCls,
                                        row: row,
                                        column: column,
                                        'natural-index': index,
                                        index: row._index,
                                        checked: this.rowChecked(row._index),
                                        disabled: this.rowDisabled(row._index),
                                        expanded: this.rowExpanded(row._index),
                                        treeNode: true,
                                        treeLevel: level
                                    },
                                    key: column._columnKey,
                                });

                                const $td = h('td', {
                                    class: this.alignCls(column, row),
                                    attrs: this.getSpan(row, column, index, colIndex)
                                }, [$tableCell]);
                                $tds.push($td);
                            }
                        });

                        // 判断节点是否展开
                        const trStyle = {};
                        if (!this.isTrShow(data._rowKey)) trStyle.display = 'none';

                        const $tableTr = h(TableTr, {
                            props: {
                                draggable: false,
                                row: row,
                                'prefix-cls': this.prefixCls
                            },
                            style: trStyle,
                            key: this.rowKey ? row._rowKey : index,
                            nativeOn: {
                                mouseenter: (e) => this.handleMouseIn(row._index, e),
                                mouseleave: (e) => this.handleMouseOut(row._index, e),
                                click: (e) => this.clickCurrentRow(row._index, e),
                                dblclick: (e) => this.dblclickCurrentRow(row._index, e)
                            }
                        }, $tds);

                        nodes.push($tableTr);

                        if (row.children && row.children.length) {
                            level++;
                            this.getChildNode(h, row, nodes, level, level);
                        }
                    });
                    return nodes;
                } else {
                    return nodes;
                }
            }
        },
        render (h) {
            let $cols = [];
            this.columns.forEach(column => {
                const $col = h('col', {
                    attrs: {
                        width: this.setCellWidth(column)
                    }
                });
                $cols.push($col);
            });
            const $colgroup = h('colgroup', {}, $cols);

            let $tableTrs = [];
            this.data.forEach((row, index) => {
                let $tds = [];

                this.columns.forEach((column, colIndex) => {
                    if (this.showWithSpan(row, column, index, colIndex)) {
                        const $tableCell = h(TableCell, {
                            props: {
                                fixed: this.fixed,
                                'prefix-cls': this.prefixCls,
                                row: row,
                                column: column,
                                'natural-index': index,
                                index: row._index,
                                checked: this.rowChecked(row._index),
                                disabled: this.rowDisabled(row._index),
                                expanded: this.rowExpanded(row._index)
                            },
                            key: column._columnKey,
                        });

                        const $td = h('td', {
                            class: this.alignCls(column, row),
                            attrs: this.getSpan(row, column, index, colIndex)
                        }, [$tableCell]);
                        $tds.push($td);
                    }
                });

                const $tableTr = h(TableTr, {
                    props: {
                        draggable: this.draggable,
                        row: row,
                        'prefix-cls': this.prefixCls
                    },
                    key: this.rowKey ? row._rowKey : index,
                    nativeOn: {
                        mouseenter: (e) => this.handleMouseIn(row._index, e),
                        mouseleave: (e) => this.handleMouseOut(row._index, e),
                        click: (e) => this.clickCurrentRow(row._index, e),
                        dblclick: (e) => this.dblclickCurrentRow(row._index, e)
                    }
                }, $tds);
                $tableTrs.push($tableTr);

                // 可展开
                if (this.rowExpanded(row._index)) {
                    const $Expand = h(Expand, {
                        props: {
                            row: row,
                            render: this.expandRender,
                            index: row._index
                        },
                        key: this.rowKey ? row._rowKey : index
                    });
                    const $td = h('td', {
                        attrs: {
                            colspan: this.columns.length
                        },
                        class: this.prefixCls + '-expanded-cell'
                    }, [$Expand]);
                    const $tr = h('tr', {
                        class: {
                            [this.prefixCls + '-expanded-hidden']: this.fixed
                        }
                    }, [$td]);
                    $tableTrs.push($tr);
                }

                // 子数据
                if (row.children && row.children.length) {
                    const $childNodes = this.getChildNode(h, row, []);
                    $childNodes.forEach(item => {
                        $tableTrs.push(item);
                    });
                }
            });

            const $tbody = h('tbody', {
                class: this.prefixCls + '-tbody'
            }, [$tableTrs]);

            return h('table', {
                attrs: {
                    cellspacing: '0',
                    cellpadding: '0',
                    border: '0'
                },
                style: this.styleObject
            }, [$colgroup, $tbody]);
        }
    };
</script>
