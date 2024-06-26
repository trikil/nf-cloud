<template>
    <div class="table-view">
        <h2>{{ header }}</h2>
        <div v-if="!result_file_not_found">
            <p v-if="description">{{ description }}</p>
            <div class="table-container">
                <table class="table table-sm table-striped">
                    <thead>
                        <tr>
                            <th @click="sort(colname)" v-for="colname in columns" :key="colname">
                                <span>{{ colname }}</span>
                                <i v-if="colname == sort_by" :class="{'fa-caret-down': sort_asc, 'fa-caret-up': !sort_asc}" class="fa ms-2"></i>
                            </th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(row, row_idx) in displayed_data" :key="`col${row_idx}`">
                            <td v-for="(content, col_idx) in row" :key="`col${col_idx}`">
                                {{content}}
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
            <Pagination
                v-if="data.length > items_per_page"
                :parent_event_bus="local_event_bus"
                :total_items="data.length"
                :page_change_event="page_change_event"
                :items_per_page="items_per_page"
            ></Pagination>
        </div>
        <div v-if="result_file_not_found">
            <p>
                {{ result_file_not_found_message }}
            </p>
        </div>
    </div>
</template>

<script>
import Vue from 'vue'
import ResultMixin from '../mixins/result.js'

const PAGE_CHANGE_EVENT = "PAGE_CHANGED"

export default {
    mixins: [
        ResultMixin
    ],
    props: {
        "path": {
            type: String,
            required: true
        },
        "header": {
            type: String,
            required: true
        },
        "description": {
            type: String,
            required: false
        },
    },
    data(){
        return {
            columns: [],                             // Table header
            data: [],                               // Table rows
            displayed_data: [],                     // Rows to be displayed
            local_event_bus: new Vue(),
            items_per_page: 50,
            page: 1,
            sort_by: "",                            // Sort column
            sort_asc: true,                         // Sort direction
            is_sorting: false                       // `true` if sorting is in progress
        }
    },
    mounted(){
        fetch(`${this.$config.nf_cloud_backend_base_url}/api/projects/${this.project_id}/table?path=${this.path}`, {
            headers: {
                "x-access-token": this.$store.state.login.jwt
            }
        }).then(response => {
            if(response.ok) {
                return response.json().then(table => {
                    this.columns = table.columns
                    this.data = table.data
                    // initial display
                    this.update_displayed_data()
                })
            } else if(response.status == 404) {
                this.internal_result_file_not_found = true
                return Promise.resolve(null)
            } else {
                return this.handleUnknownResponse(response)
            }
        }).finally(() => {
            this.result_file_loading = false
        })
        this.local_event_bus.$on(PAGE_CHANGE_EVENT, new_page => this.page = new_page)
    },
    methods: {
        /**
         * Sort rows by given column. If the given column is already selected the sort is reversed.
         * @param  {String} new_sort_by Column name to be sorted
         */
        sort(new_sort_by){
            if(!this.is_sorting){
                this.is_sorting = true
                // Sort new if sort column changed
                if(this.sort_by != new_sort_by){
                    this.sort_by = new_sort_by
                    var sort_column = this.header.indexOf(this.sort_by)
                    // Sort array ascending
                    this.data = this.data.sort((elem_x, elem_y) => {
                        if(elem_x[sort_column] < elem_y[sort_column]){
                            return -1
                        } else if(elem_x[sort_column] > elem_y[sort_column]) {
                            return 1
                        } else {
                            return 0
                        }
                    });
                    // Reverse array if sorting is desc
                    if(!this.sort_asc) this.data = this.data.reverse()
                } else {
                    // Just reverse if column is the same
                    this.sort_asc = !this.sort_asc
                    this.data = this.data.reverse()
                }
                this.update_displayed_data()
                this.is_sorting = false
            }
        },
        /**
         * Updates the displayed rows according the selected page.
         */
        update_displayed_data(){
            let start = (this.page - 1) * this.items_per_page
            this.displayed_data = this.data.slice(
                start,
                start + this.items_per_page
            )
        }
    },
    computed: {
        page_change_event(){
            return PAGE_CHANGE_EVENT
        }
    },
    watch: {
        page(){
            this.update_displayed_data()
        }
    }
}
</script>