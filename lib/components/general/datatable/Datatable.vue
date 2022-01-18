<template>
    <div>
        <div class="intro-y box mt-2">
            <div class="mb-2 intro-y flex justify-between items-center">
                <div class="flex px-4 mt-2">
                    <div class="input-group">
                    <div id="input-group-search" class="input-group-text">
                        <SearchIcon class="w-4 h-4" />
                    </div>
                    <input type="text" class="form-control" v-model="search" @keyup="onSearch()" :placeholder="text.search" :aria-label="text.search" aria-describedby="input-group-search" />
                </div>
                </div>
                <div>
                    <div class="rounded flex mr-4">
                        <span class="mt-4 mr-4"><span v-html="text.show_entries"></span></span>
                        <select class="form-select mt-2 sm:mr-2" aria-label="Registros de la tabla" v-model="limit" v-on:change="getData(); changePage(1);">
                            <option  v-for="entry in showEntries" :key="entry" :value="entry">{{ entry }}</option>
                        </select>
                    </div>
                </div>
		    </div>
            <div class="overflow-x-auto">
                <table class="table">
                    <thead>
                        <tr class="bg-theme-23 dark:bg-dark-2">
                            
                            <th v-for="column in getColumns" :key="column" id="{{ column.name }}" class="border-b-2 dark:border-dark-5 whitespace-nowrap cursor-pointer" v-on:click="sortColumns(column.name)">
                                <div class="flex justify-between ">
                                    {{ column.text }}
                                    <span v-if="sortColumn && sortColumn === column.name ">
                                        <ChevronDownIcon class="w-4 h-4 ml-auto sm:ml-2" v-if="sortDirection === 'desc'" />  
                                        <ChevronUpIcon class="w-4 h-4 ml-auto sm:ml-2" v-else />       
                                    </span>
                                     
                                </div>
                            </th>
                        </tr>
                    </thead>
                    <tbody>
                    <tr v-for="(row,index) in getRows" :key="index" :class="{ 'bg-gray-100 dark:bg-dark-5' : isOdd(index) }">
                        <td v-for="column in getColumns" :key="column" class="border-b dark:border-dark-5">{{ row[column.name] }}</td>  
                    </tr>
                    <tr v-show="!getRows.length">
                        <td  class="text-center"  :colspan="this.columns.length + 1"><span v-html="text.empty"></span></td>
                    </tr>
                    </tbody>
                </table>
            </div>  
            <div class="intro-y flex justify-center md:justify-between items-center p-3" v-if="this.$route.query.page <= this.totalPages || !this.$route.query.page">
                <div class="hidden md:flex" v-if="totalPages > 1"><span v-html="text.showing"></span> <strong class="mx-2">{{ pageStart + 1 }}</strong> / <strong class="mx-2">{{ pageEnd }}</strong> <span v-html="text.of"></span> <strong class="mx-2">{{ totalRows }}</strong> <span v-html="text.records"></span> </div>
                <div class="hidden md:flex" v-else><span v-html="text.showing"></span> {{ totalRows }} <span v-html="text.records"></span></div>
                <ul class="mt-2 hidden md:flex">
                    <li class="mx-1  rounded-lg">
                        <a class="flex items-center  disabled px-3 py-2 rounded" v-on:click="previousPage()" v-bind:class="checkPreviousPage()" href="javascript:void(0);">
                            <span class="mx-1 "><span v-html="text.previous"></span></span>
                        </a>
                    </li>
                    <li v-for="page in displayPages" :key="page" class="mx-1 rounded-lg invisible md:visible">
                        <a class="flex font-bold  active px-3 py-2 rounded" v-on:click="changePage(page)" v-bind:class="isCurrentPage(page)" href="javascript:void(0);">{{ page }}</a>
                    </li>
                   
                    <li class="mx-1 rounded-lg">
                        <a class="flex items-center px-3 py-2 rounded" v-on:click="nextPage()" v-bind:class="checkNextPage()" href="javascript:void(0);">
                            <span class="mx-1"><span v-html="text.next"></span></span>
                        </a>
                    </li>
                </ul>
                 <ul class="flex mt-2 md:hidden">
                    <li class="mx-1  rounded-lg">
                        <a class="flex items-center  disabled px-3 py-2 rounded" v-on:click="previousPage()" v-bind:class="checkPreviousPage()" href="javascript:void(0);">
                            <span class="mx-1 "><span v-html="text.previous"></span></span>
                        </a>
                    </li>
                   
                    <li class="mx-1 rounded-lg">
                        <a class="flex items-center px-3 py-2 rounded" v-on:click="nextPage()" v-bind:class="checkNextPage()" href="javascript:void(0);">
                            <span class="mx-1"><span v-html="text.next"></span></span>
                        </a>
                    </li>
                </ul>
            
            </div>
        </div>
    </div>
</template>

<script>
import { defineComponent } from 'vue'
import axios from "axios"

export default defineComponent({
    data () {
        return {
            showEntries : [10,25,50,100],
            rows : [],
            limit : 10,
            currentPage : parseInt(this.$route.query.page) || 1,
            paginatedRows : [],
            pageStart : 0,
            pageEnd : 0,
            totalRows : 0,
            sortColumn :  '',
            sortDirection : 'desc',
            search : '',
            awaitingSearch: false,
            text : []
        }
    },
    mounted () {
        this.initializeComponent()
        this.getData()
    },
    props : {
        columns : Array,
        endpoint : String,
        searchText : String,
        showEntriesText : String,
        showingText : String,
        ofText : String,
        emptyText : String,
        recordsText : String,
        previousText : String,
        nextText : String
    },
    computed : {
        getColumns () {
            return this.columns || []
        },
        getRows () {
            return this.rows || []
        },
        displayPages() {
            let totalPages = this.totalPages;
            let currentPage = this.currentPage;
            let arrayLen = (currentPage > 99) ? 3 : 5;
            if ([1, 2].includes(currentPage)) currentPage = 3;
            else if ([totalPages -1, totalPages].includes(currentPage)) currentPage = totalPages - Math.trunc(arrayLen / 2);
            return [...Array(arrayLen).keys()].map(i => i - Math.trunc(arrayLen / 2) + currentPage)
        },
        totalPages () {
            return Math.ceil(this.totalRows / this.limit)
        }
    },
    methods : {
        initializeComponent () {
            this.setText()
        },
        async getData () {
            await axios.get(
                    this.endpoint,
                    {
                        params : {
                            page : this.currentPage,
                            entries : this.limit,
                            sortColumn : this.sortColumn,
                            sortDirection : this.sortDirection,
                            search: this.search
                        }
                    }
            ).then((response) =>{
                this.totalRows = response.data.total
                this.pageStart = (this.currentPage * this.limit) - this.limit 
                this.pageEnd = this.currentPage * this.limit
                this.rows = response.data.data
            })
        },
        setText() {
            this.text = {
                "search" : this.searchText || "Buscar...",
                "show_entries" : this.showEntriesText || "Mostrar",
                "showing" : this.showingText || "Mostrando",
                "of" : this.ofText || "de",
                "records" : this.recordsText || "registro(s)",
                "previous" : this.previousText || "Anterior",
                "next" : this.nextText || "Siguiente",
                "empty" : this.emptyText || "No hay registros para mostrar"
            }    
        },
        onSearch () {
            if (!this.awaitingSearch) {
                setTimeout(() => {
                    this.changePage(1)
                    this.getData();
                    this.awaitingSearch = false;
                }, 1000); // 1 sec delay
            }
            this.awaitingSearch = true;
        },
        sortColumns (index) {
            console.log(this.sortColumn,index)
            if(this.sortColumn !== index){
                this.sortColumn = index  
            }
            
            this.sortDirection = (this.sortDirection === 'asc') ? 'desc' : 'asc'    
            
            this.getData()
        },
        isOdd (value) {
            return value % 2;
        },
        changePage (page) {
            this.currentPage = page
            this.getData()
            this.$router.push({ path : this.$route.path, query : {
                page : this.currentPage
            }})
        },
        previousPage() {
            if(this.currentPage > 1) {
                this.currentPage--
            }
            this.changePage(this.currentPage)
        },
        nextPage() {
            if( this.currentPage  < this.totalPages) {
                this.currentPage++
            }

            this.changePage(this.currentPage)
        },
        isCurrentPage(page) {
            if(page <= this.totalPages) {
                if(page == this.currentPage) {
                    if(this.totalPages === 1) {
                        return 'btn-secondary ml-5'
                    }
                    return 'btn-primary'
                } else {
                    return 'btn-secondary';
                }
            } else {
                return 'hidden'
            }
            
        },
        checkPreviousPage()
        {
            if(this.currentPage > 1) {
                return 'font-bold btn-secondary'
            } else {
                return 'cursor-text'
            }
        },
        checkNextPage()
        {
            if(this.currentPage < this.totalPages) {
                 return 'font-bold btn-secondary'
            } else {
                return 'cursor-text'
            }
        }
    }
})
</script>
