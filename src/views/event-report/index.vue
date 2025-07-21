<template>
  <div class="event-report-container">
    <!-- Header Title -->
    <div class="page-header">
      <h1 class="page-title">重点事件通报</h1>
    </div>

    <!-- Statistics Section -->
    <div class="statistics-section">
      <div class="statistics-card">
        <div class="statistics-content">
          <i class="el-icon-message statistics-icon" />
          <div class="statistics-text">
            <span class="statistics-count">共12,580条重点事件</span>
          </div>
        </div>
        <el-button type="primary" class="custom-rules-btn">
          自定义通报规则
        </el-button>
      </div>
    </div>

    <!-- Filter Section -->
    <div class="filter-section">
      <el-row :gutter="20" class="filter-row">
        <el-col :span="4">
          <el-select v-model="filters.eventType" placeholder="事件类型" clearable>
            <el-option label="全部类型" value="" />
            <el-option label="数据泄露" value="data_leak" />
            <el-option label="系统入侵" value="system_intrusion" />
            <el-option label="恶意攻击" value="malicious_attack" />
            <el-option label="权限滥用" value="privilege_abuse" />
          </el-select>
        </el-col>
        <el-col :span="4">
          <el-select v-model="filters.severity" placeholder="严重程度" clearable>
            <el-option label="全部级别" value="" />
            <el-option label="高" value="high" />
            <el-option label="中" value="medium" />
            <el-option label="低" value="low" />
          </el-select>
        </el-col>
        <el-col :span="4">
          <el-select v-model="filters.status" placeholder="处理状态" clearable>
            <el-option label="全部状态" value="" />
            <el-option label="待处理" value="pending" />
            <el-option label="处理中" value="processing" />
            <el-option label="已处理" value="resolved" />
            <el-option label="已关闭" value="closed" />
          </el-select>
        </el-col>
        <el-col :span="4">
          <el-date-picker
            v-model="filters.timeRange"
            type="daterange"
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            clearable
          />
        </el-col>
        <el-col :span="6">
          <el-input
            v-model="filters.search"
            placeholder="模糊搜索事件标题或描述"
            clearable
            @input="handleSearch"
          >
            <i slot="prefix" class="el-input__icon el-icon-search" />
          </el-input>
        </el-col>
        <el-col :span="2">
          <el-button type="primary" @click="handleFilter">搜索</el-button>
        </el-col>
      </el-row>
    </div>

    <!-- Event Cards Grid -->
    <div class="events-grid">
      <el-row :gutter="20">
        <el-col
          v-for="event in filteredEvents"
          :key="event.id"
          :span="8"
          class="event-card-col"
        >
          <div class="event-card">
            <div class="event-header">
              <i class="el-icon-folder event-folder-icon" />
              <el-tag
                :type="getSeverityTagType(event.severity)"
                size="small"
                class="event-type-tag"
              >
                [{{ getSeverityLabel(event.severity) }}]
              </el-tag>
            </div>

            <div class="event-title">
              {{ event.title }}
            </div>

            <div class="event-time">
              <i class="el-icon-time" />
              {{ formatTime(event.createTime) }}
            </div>

            <div class="event-description">
              {{ event.description }}
            </div>

            <div class="event-data-volume">
              <i class="el-icon-data-analysis" />
              涉及数据量：{{ event.dataVolume }}
            </div>

            <div class="event-tags">
              <el-tag
                :type="getSeverityTagType(event.severity)"
                size="mini"
              >
                {{ getSeverityLabel(event.severity) }}
              </el-tag>
              <el-tag
                :type="getStatusTagType(event.status)"
                size="mini"
              >
                {{ getStatusLabel(event.status) }}
              </el-tag>
            </div>

            <div class="event-actions">
              <el-button
                type="text"
                size="small"
                @click="handleViewDetail(event)"
              >
                查看详情/处理
              </el-button>
            </div>
          </div>
        </el-col>
      </el-row>
    </div>

    <!-- Pagination -->
    <div class="pagination-section">
      <el-pagination
        background
        layout="total, sizes, prev, pager, next, jumper"
        :current-page="pagination.currentPage"
        :page-sizes="[9, 18, 36, 54]"
        :page-size="pagination.pageSize"
        :total="pagination.total"
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
      />
    </div>
  </div>
</template>

<script>
export default {
  name: 'EventReport',
  data() {
    return {
      filters: {
        eventType: '',
        severity: '',
        status: '',
        timeRange: null,
        search: ''
      },
      pagination: {
        currentPage: 1,
        pageSize: 9,
        total: 0
      },
      events: [],
      loading: false
    }
  },
  computed: {
    allFilteredEvents() {
      let filtered = this.events

      // Apply filters
      if (this.filters.eventType) {
        filtered = filtered.filter(event => event.type === this.filters.eventType)
      }
      if (this.filters.severity) {
        filtered = filtered.filter(event => event.severity === this.filters.severity)
      }
      if (this.filters.status) {
        filtered = filtered.filter(event => event.status === this.filters.status)
      }
      if (this.filters.search) {
        const searchTerm = this.filters.search.toLowerCase()
        filtered = filtered.filter(event =>
          event.title.toLowerCase().includes(searchTerm) ||
          event.description.toLowerCase().includes(searchTerm)
        )
      }

      return filtered
    },
    filteredEvents() {
      const filtered = this.allFilteredEvents

      // Apply pagination
      const start = (this.pagination.currentPage - 1) * this.pagination.pageSize
      const end = start + this.pagination.pageSize
      return filtered.slice(start, end)
    }
  },
  watch: {
    allFilteredEvents: {
      handler(newVal) {
        this.pagination.total = newVal.length
      },
      immediate: true
    }
  },
  created() {
    this.loadEvents()
  },
  methods: {
    loadEvents() {
      this.loading = true
      // Mock data
      this.events = [
        {
          id: 1,
          type: 'data_leak',
          title: '车辆定位数据存在大规模未经授权传输风险',
          description: '检测到车辆GPS定位系统存在数据泄露风险，可能导致用户隐私信息泄露',
          createTime: '2025-07-18 10:15:30',
          severity: 'high',
          status: 'pending',
          dataVolume: '150,000条高精度定位数据'
        },
        {
          id: 2,
          type: 'system_intrusion',
          title: '核心业务系统检测到异常登录行为',
          description: '系统监控发现多个异常IP地址尝试登录核心业务系统，存在安全风险',
          createTime: '2025-07-18 09:42:15',
          severity: 'high',
          status: 'processing',
          dataVolume: '500个异常登录尝试'
        },
        {
          id: 3,
          type: 'malicious_attack',
          title: '网络防火墙拦截大量恶意攻击请求',
          description: '防火墙系统在过去24小时内拦截了大量来自境外的恶意攻击请求',
          createTime: '2025-07-18 08:30:45',
          severity: 'medium',
          status: 'resolved',
          dataVolume: '1,200次恶意请求'
        },
        {
          id: 4,
          type: 'privilege_abuse',
          title: '员工权限使用异常检测报告',
          description: '检测到某员工账户在非工作时间频繁访问敏感数据，存在权限滥用风险',
          createTime: '2025-07-17 22:18:20',
          severity: 'medium',
          status: 'pending',
          dataVolume: '200次敏感数据访问'
        },
        {
          id: 5,
          type: 'data_leak',
          title: '客户个人信息数据库访问异常',
          description: '发现客户个人信息数据库存在异常访问记录，需要立即进行安全评估',
          createTime: '2025-07-17 15:25:10',
          severity: 'high',
          status: 'processing',
          dataVolume: '50,000条客户信息'
        },
        {
          id: 6,
          type: 'system_intrusion',
          title: '服务器文件完整性检查发现异常',
          description: '系统文件完整性检查发现多个核心文件被非授权修改',
          createTime: '2025-07-17 13:45:30',
          severity: 'high',
          status: 'resolved',
          dataVolume: '15个系统文件'
        },
        {
          id: 7,
          type: 'malicious_attack',
          title: 'DDoS攻击防护系统告警',
          description: '检测到针对公司官网的分布式拒绝服务攻击，防护系统已自动启动',
          createTime: '2025-07-17 11:30:15',
          severity: 'low',
          status: 'resolved',
          dataVolume: '10GB攻击流量'
        },
        {
          id: 8,
          type: 'privilege_abuse',
          title: '管理员权限使用审计异常',
          description: '管理员权限使用审计中发现多个异常操作记录，需要进一步调查',
          createTime: '2025-07-17 09:15:45',
          severity: 'medium',
          status: 'pending',
          dataVolume: '30次权限操作'
        },
        {
          id: 9,
          type: 'data_leak',
          title: '数据传输加密异常检测',
          description: '监控系统发现部分数据传输过程中加密机制失效，存在数据泄露风险',
          createTime: '2025-07-16 16:20:30',
          severity: 'medium',
          status: 'closed',
          dataVolume: '2,500条传输记录'
        },
        {
          id: 10,
          type: 'system_intrusion',
          title: '内网安全扫描发现漏洞',
          description: '定期安全扫描发现内网多个系统存在高危安全漏洞，需要紧急修复',
          createTime: '2025-07-16 14:10:20',
          severity: 'high',
          status: 'processing',
          dataVolume: '25个安全漏洞'
        },
        {
          id: 11,
          type: 'malicious_attack',
          title: '恶意邮件攻击防护告警',
          description: '邮件安全系统拦截到大量包含恶意附件的钓鱼邮件',
          createTime: '2025-07-16 10:30:45',
          severity: 'low',
          status: 'resolved',
          dataVolume: '300封恶意邮件'
        },
        {
          id: 12,
          type: 'data_leak',
          title: '第三方API数据访问异常',
          description: '监控发现第三方合作伙伴API存在异常数据访问行为',
          createTime: '2025-07-16 08:45:15',
          severity: 'medium',
          status: 'pending',
          dataVolume: '80,000次API调用'
        }
      ]
      this.pagination.total = this.events.length
      this.loading = false
    },

    handleFilter() {
      this.pagination.currentPage = 1
    },

    handleSearch() {
      this.pagination.currentPage = 1
    },

    handleSizeChange(val) {
      this.pagination.pageSize = val
      this.pagination.currentPage = 1
    },

    handleCurrentChange(val) {
      this.pagination.currentPage = val
    },

    handleViewDetail(event) {
      this.$message.info(`查看事件详情: ${event.title}`)
    },

    getSeverityLabel(severity) {
      const labels = {
        high: '高危',
        medium: '中危',
        low: '低危'
      }
      return labels[severity] || severity
    },

    getSeverityTagType(severity) {
      const types = {
        high: 'danger',
        medium: 'warning',
        low: 'info'
      }
      return types[severity] || 'info'
    },

    getStatusLabel(status) {
      const labels = {
        pending: '待处理',
        processing: '处理中',
        resolved: '已处理',
        closed: '已关闭'
      }
      return labels[status] || status
    },

    getStatusTagType(status) {
      const types = {
        pending: 'warning',
        processing: 'primary',
        resolved: 'success',
        closed: 'info'
      }
      return types[status] || 'info'
    },

    formatTime(timeStr) {
      return timeStr
    }
  }
}
</script>

<style lang="scss" scoped>
.event-report-container {
  padding: 20px;
  background-color: #f5f7fa;
  min-height: 100vh;
}

.page-header {
  margin-bottom: 20px;

  .page-title {
    color: #409EFF;
    font-size: 28px;
    font-weight: bold;
    margin: 0;
  }
}

.statistics-section {
  margin-bottom: 24px;

  .statistics-card {
    background: white;
    border-radius: 8px;
    padding: 20px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    display: flex;
    justify-content: space-between;
    align-items: center;

    .statistics-content {
      display: flex;
      align-items: center;

      .statistics-icon {
        font-size: 32px;
        color: #409EFF;
        margin-right: 16px;
      }

      .statistics-text {
        .statistics-count {
          font-size: 18px;
          font-weight: bold;
          color: #303133;
        }
      }
    }

    .custom-rules-btn {
      height: 40px;
      padding: 0 24px;
    }
  }
}

.filter-section {
  margin-bottom: 24px;
  background: white;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);

  .filter-row {
    .el-select,
    .el-date-editor,
    .el-input {
      width: 100%;
    }
  }
}

.events-grid {
  margin-bottom: 24px;

  .event-card-col {
    margin-bottom: 20px;
  }

  .event-card {
    background: white;
    border-radius: 8px;
    padding: 20px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    height: 320px;
    display: flex;
    flex-direction: column;
    transition: box-shadow 0.3s ease;

    &:hover {
      box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
    }

    .event-header {
      display: flex;
      align-items: center;
      margin-bottom: 12px;

      .event-folder-icon {
        font-size: 18px;
        color: #909399;
        margin-right: 8px;
      }

      .event-type-tag {
        font-weight: bold;
      }
    }

    .event-title {
      font-size: 16px;
      font-weight: bold;
      color: #303133;
      margin-bottom: 8px;
      line-height: 1.4;
      display: -webkit-box;
      -webkit-line-clamp: 2;
      -webkit-box-orient: vertical;
      overflow: hidden;
    }

    .event-time {
      font-size: 14px;
      color: #909399;
      margin-bottom: 12px;
      display: flex;
      align-items: center;

      i {
        margin-right: 4px;
      }
    }

    .event-description {
      font-size: 14px;
      color: #606266;
      line-height: 1.5;
      margin-bottom: 12px;
      flex: 1;
      display: -webkit-box;
      -webkit-line-clamp: 3;
      -webkit-box-orient: vertical;
      overflow: hidden;
    }

    .event-data-volume {
      font-size: 14px;
      color: #E6A23C;
      margin-bottom: 12px;
      display: flex;
      align-items: center;

      i {
        margin-right: 4px;
      }
    }

    .event-tags {
      margin-bottom: 12px;

      .el-tag {
        margin-right: 8px;
      }
    }

    .event-actions {
      text-align: right;

      .el-button {
        color: #409EFF;
        font-weight: 500;

        &:hover {
          color: #66b1ff;
        }
      }
    }
  }
}

.pagination-section {
  background: white;
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  text-align: right;
}

// Responsive design
@media (max-width: 1200px) {
  .events-grid {
    .event-card-col {
      span: 12; // 2 cards per row on medium screens
    }
  }
}

@media (max-width: 768px) {
  .events-grid {
    .event-card-col {
      span: 24; // 1 card per row on small screens
    }
  }

  .filter-section {
    .filter-row {
      .el-col {
        margin-bottom: 12px;
      }
    }
  }
}
</style>
