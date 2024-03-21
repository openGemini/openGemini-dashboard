<template>
  <div>
    <!-- 第一个卡片 - 查询条件 -->
    <el-card shadow="always" class="card-wrapper">
      <el-form :model="formData" ref="form" label-width="90px">
        <div class="top-container">
          <div class="input-container">
            <el-form-item>
              <el-select v-model="formData.timeRange" @change="handleSelectChange" clearable placeholder="Time range">
                <el-option label="Recent 30 min" value="Recent 30 min" />
                <el-option label="Recent 1 hour" value="Recent 1 hour" />
                <el-option label="Recent 1 week" value="Recent 1 week" />
                <el-option label="Recent 1 month" value="Recent 1 month" />
                <!-- Add more options as needed -->
              </el-select>
            </el-form-item>
          </div>
          <div class="input-container">
            <el-form-item>
              <el-select
                v-model="formData.selectedDatabase"
                @change="handleSelectChange"
                clearable
                placeholder="All databases"
              >
                <el-option label="All databases" value="all" />
                <el-option v-for="database in allDatabases" :label="database" :value="database" :key="database" />
              </el-select>
            </el-form-item>
          </div>
          <div class="input-container">
            <el-form-item>
              <el-select v-model="formData.rowLimit" @change="handleSelectChange" clearable placeholder="Limit rows">
                <el-option label="Limit 50" value="Limit 50" />
                <el-option label="Limit 100" value="Limit 100" />
                <el-option label="Limit 500" value="Limit 500" />
                <!-- Add more options as needed -->
              </el-select>
            </el-form-item>
          </div>
          <div class="input-search">
            <el-form-item>
              <el-input v-model="formData.filterKeyword" placeholder="Filter keyword" clearable />
            </el-form-item>
          </div>
          <div class="query-button">
            <el-form-item>
              <el-button type="primary" @click="executeQuery">查询</el-button>
            </el-form-item>
          </div>
        </div>
      </el-form>
    </el-card>

    <!-- 第二个卡片 - 慢查询展示 -->
    <el-card shadow="always" class="card-wrapper">
      <el-table
        :data="displayedQueries"
        :header-cell-style="{ background: '#FFFFFF', color: '#606266' }"
        @sort-change="fetchData"
        border
        style="width: 100%"
      >
        <el-table-column :prop="item" :label="item" v-for="item in queryHeader" :key="item" />
      </el-table>
      <!-- 分页 -->
      <el-pagination
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
        :current-page="currentPage"
        :page-sizes="[25, 50, 100, 200]"
        :page-size="pageSize"
        :total="queryData.length"
        layout="sizes, prev, pager, next, jumper"
        style="float: right"
      />
    </el-card>
  </div>
</template>

<script lang="ts">
import axios from "axios"

interface FormData {
  timeRange: string
  selectedDatabase: string
  rowLimit: string
  filterKeyword: string
}

export default {
  name: "LogTable",
  data() {
    return {
      // ... existing data properties ...

      // New properties for slow queries
      queryHeader: [],
      queryData: [],
      displayedQueries: [],
      currentPage: 1,
      pageSize: 25,
      formData: {
        // ... existing form data ...
        timeRange: "Recent 30 min", // Default time range
        selectedDatabase: "all", //Default selected database
        rowLimit: "Limit 50", // Default row limit
        filterKeyword: ""
      } as FormData,
      allDatabases: []
    }
  },
  created() {
    this.fetchData()
  },
  methods: {
    fetchData() {
      // ... existing fetchData method ...

      // Additional code for slow queries
      const slowQueryApiUrl = "http://127.0.0.1:8086/query"
      let slowQueryDB
      if (this.formData.selectedDatabase === "all") {
        slowQueryDB = ""
      } else {
        slowQueryDB = this.formData.selectedDatabase
      }
      const slowQuerySql = "show queries"

      const databaseQuerySql = "show databases"

      axios
        .get(slowQueryApiUrl, {
          params: {
            pretty: true,
            q: databaseQuerySql
          }
        })
        .then((response: any) => {
          const databaseResults = response.data.results[0]
          console.log(databaseResults)
          if (databaseResults && databaseResults.series) {
            this.allDatabases = databaseResults.series[0].values.map((value: any) => value[0])
          }
        })
        .catch((error) => {
          console.error("Error fetching databases:", error)
        })

      axios
        .get(slowQueryApiUrl, {
          params: {
            db: slowQueryDB,
            pretty: true,
            q: slowQuerySql
          }
        })
        .then((response) => {
          const results = response.data.results[0]
          if (results && results.series) {
            this.queryHeader = results.series[0].columns
            this.queryData = results.series[0].values.map((row: any[]) => {
              return results.series[0].columns.reduce((rowData: any, columnName: string, index: number) => {
                rowData[columnName] = row[index]
                return rowData
              }, {})
            })
            this.updateDisplayedQueries()
          } else {
            console.error("No series or empty series in slow queries results.")
          }
        })
        .catch((error) => {
          console.error("Error fetching slow queries:", error)
        })
    },
    // ... existing methods ...

    handleSelectChange() {
      // 任一选择框改变后立即出发查询筛选操作
      this.fetchData()
    },

    updateDisplayedQueries() {
      const startIndex = (this.currentPage - 1) * this.pageSize
      this.displayedQueries = this.queryData.slice(startIndex, startIndex + this.pageSize)
    },
    handleSizeChange(newSize: number) {
      this.pageSize = newSize
      this.currentPage = 1
      this.updateDisplayedQueries()
    },
    handleCurrentChange(newPage: number) {
      this.currentPage = newPage
      this.updateDisplayedQueries()
    },
    executeQuery() {
      this.fetchData()
    }
    // ... existing methods ...
  }
  // ... existing lifecycle hooks ...
}
</script>

<style scoped>
/* ... existing styles ... */

.card-wrapper {
  margin-bottom: 20px;
  :deep(.el-card__body) {
    padding-bottom: 2px;
  }
}

.top-container {
  display: flex;
}

.input-container {
  width: 250px;
}

.input-search {
  width: 350px;
}

.query-button {
  margin-left: -10px;
}

.input-search,
.query-button {
  display: inline-block;
  margin: 0;
  padding: 0;
}
</style>
