<template>
    <top-nav-bar v-if="!embed" :title="routeInfo.title" />
    <section v-bind="$attrs" :class="{'container': !embed}" class="log-panel">
        <div class="log-content">
            <data-table @page-changed="onPageChanged" ref="dataTable" :total="total" :size="pageSize" :page="pageNumber" :embed="embed">
                <template #navbar v-if="!embed || showFilters">
                    <el-form-item>
                        <search-field />
                    </el-form-item>
                    <el-form-item>
                        <namespace-select
                            data-test-id="logs-namespace-selector"
                            data-type="flow"
                            v-if="$route.name !== 'flows/update'"
                            :value="$route.query.namespace"
                            @update:model-value="onDataTableValue('namespace', $event)"
                        />
                    </el-form-item>
                    <el-form-item>
                        <log-level-selector
                            data-test-id="logs-log-level-selector"
                            :value="selectedLogLevel"
                            @update:model-value="onDataTableValue('level', $event)"
                        />
                    </el-form-item>
                    <el-form-item>
                        <date-filter
                            @update:is-relative="onDateFilterTypeChange"
                            @update:filter-value="onDataTableValue"
                        />
                    </el-form-item>
                    <el-form-item>
                        <el-switch
                            :model-value="showChart"
                            @update:model-value="onShowChartChange"
                            :active-text="$t('show chart')"
                        />
                    </el-form-item>
                    <el-form-item>
                        <filters :storage-key="storageKeys.LOGS_FILTERS" />
                    </el-form-item>
                    <el-form-item>
                        <refresh-button class="float-right" @refresh="refresh" />
                    </el-form-item>
                </template>

                <template v-if="showStatChart()" #top>
                    <el-card shadow="never" class="mb-3" v-loading="!statsReady">
                        <div>
                            <template v-if="hasStatsData">
                                <Logs :data="logDaily" />
                            </template>
                            <template v-else>
                                <el-alert type="info" :closable="false" class="m-0">
                                    {{ $t('no result') }}
                                </el-alert>
                            </template>
                        </div>
                    </el-card>
                </template>

                <template #table>
                    <div v-loading="isLoading">
                        <div v-if="logs === undefined || logs.length === 0">
                            <el-alert type="info" :closable="false" class="text-muted">
                                {{ $t('no result') }}
                            </el-alert>
                        </div>
                        <div v-else class="logs-wrapper">
                            <template v-for="(log, i) in logs" :key="`${log.taskRunId}-${i}`">
                                <log-line
                                    level="TRACE"
                                    filter=""
                                    :exclude-metas="isFlowEdit ? ['namespace', 'flowId'] : []"
                                    :log="log"
                                />
                            </template>
                        </div>
                    </div>
                </template>
            </data-table>
        </div>
    </section>
</template>

<script>
    import LogLine from "../logs/LogLine.vue";
    import {mapState} from "vuex";
    import RouteContext from "../../mixins/routeContext";
    import TopNavBar from "../../components/layout/TopNavBar.vue";
    import RestoreUrl from "../../mixins/restoreUrl";
    import DataTableActions from "../../mixins/dataTableActions";
    import NamespaceSelect from "../namespace/NamespaceSelect.vue";
    import SearchField from "../layout/SearchField.vue";
    import DateFilter from "../executions/date-select/DateFilter.vue";
    import LogLevelSelector from "./LogLevelSelector.vue";
    import DataTable from "../../components/layout/DataTable.vue";
    import RefreshButton from "../../components/layout/RefreshButton.vue";
    import _merge from "lodash/merge";
    import Logs from "../dashboard/components/charts/logs/Bar.vue";
    import Filters from "../saved-filters/Filters.vue";
    import {storageKeys} from "../../utils/constants";
    

    export default {
        mixins: [RouteContext, RestoreUrl, DataTableActions],
        components: {
            Filters,
            DataTable, LogLine, NamespaceSelect, DateFilter, SearchField, LogLevelSelector, RefreshButton, TopNavBar, Logs},
        props: {
            logLevel: {
                type: String,
                default: undefined
            },
            embed: {
                type: Boolean,
                default: false
            },
            charts: {
                type: Boolean,
                default: true
            },
            showFilters: {
                type: Boolean,
                default: false
            },
            filters: {
                type: Object,
                default: null
            },
        },
        data() {
            return {
                isDefaultNamespaceAllow: true,
                task: undefined,
                isLoading: false,
                refreshDates: false,
                statsReady: false,
                statsData: [],
                canAutoRefresh: false,
                showChart: ["true", null].includes(localStorage.getItem(storageKeys.SHOW_LOGS_CHART)),
            };
        },
        computed: {
            storageKeys() {
                return storageKeys
            },
            ...mapState("log", ["logs", "total", "level"]),
            ...mapState("stat", ["logDaily"]),
            routeInfo() {
                return {
                    title: this.$t("logs"),
                };
            },
            isFlowEdit() {
                return this.$route.name === "flows/update"
            },
            isNamespaceEdit() {
                return this.$route.name === "namespaces/update"
            },
            selectedLogLevel() {
                return this.logLevel || this.$route.query.level || localStorage.getItem("defaultLogLevel") || "INFO";
            },
            endDate() {
                if (this.$route.query.endDate) {
                    return this.$route.query.endDate;
                }
                return undefined;
            },
            startDate() {
                this.refreshDates;
                if (this.$route.query.startDate) {
                    return this.$route.query.startDate;
                }
                if (this.$route.query.timeRange) {
                    return this.$moment().subtract(this.$moment.duration(this.$route.query.timeRange).as("milliseconds")).toISOString(true);
                }

                // the default is PT30D
                return this.$moment().subtract(7, "days").toISOString(true);
            },
            namespace() {
                return this.$route.params.namespace ?? this.$route.params.id;
            },
            flowId() {
                return this.$route.params.id;
            },
            countStats() {
                return [...this.logDaily || []].reduce((a, b) => {
                    return a + Object.values(b.counts).reduce((a, b) => a + b, 0);
                }, 0);
            },
            hasStatsData() {
                return this.countStats > 0;
            },
        },
        beforeRouteEnter(to, from, next) {
            const defaultNamespace = localStorage.getItem(storageKeys.DEFAULT_NAMESPACE);
            const query = {...to.query};
            if (defaultNamespace) {
                query.namespace = defaultNamespace; 
            }
            next(vm => {
                vm.$router?.replace({query});
            });
        },
        methods: {
            onDateFilterTypeChange(event) {
                this.canAutoRefresh = event;
            },
            showStatChart() {
                return this.charts && this.showChart;
            },
            onShowChartChange(value) {
                this.showChart = value;
                localStorage.setItem(storageKeys.SHOW_LOGS_CHART, value);
                if (this.showStatChart()) {
                    this.loadStats();
                }
            },
            refresh() {
                this.refreshDates = !this.refreshDates;
                this.load();
            },
            loadQuery(base) {
                // eslint-disable-next-line no-unused-vars
                const {triggerId, ...rest} = this.filters || {};
                let queryFilter = this.filters ?? this.queryWithFilter();

                if (this.isFlowEdit) {
                    queryFilter["namespace"] = this.namespace;
                    queryFilter["flowId"] = this.flowId;
                } else if (this.isNamespaceEdit) {
                    queryFilter["namespace"] = this.namespace;
                }

                if (!queryFilter["startDate"] || !queryFilter["endDate"]) {
                    queryFilter["startDate"] = this.startDate;
                    queryFilter["endDate"] = this.endDate;
                }

                delete queryFilter["level"];

                return _merge(base, queryFilter)
            },
            load() {
                this.isLoading = true

                // eslint-disable-next-line no-unused-vars
                const data = {
                    page: this.filters ? this.internalPageNumber : this.$route.query.page || this.internalPageNumber,
                    size: this.filters ? this.internalPageSize : this.$route.query.size || this.internalPageSize,
                    ...this.filters
                };
                this.$store
                    .dispatch("log/findLogs", this.loadQuery({
                        ...data,
                        minLevel: this.filters ? null : this.selectedLogLevel,
                        sort: "timestamp:desc"
                    }))
                    .finally(() => {
                        this.isLoading = false
                        this.saveRestoreUrl();
                    });

                this.loadStats();
            },
            loadStats() {
                this.statsReady = false;
                this.$store
                    .dispatch("stat/logDaily", this.loadQuery({
                        startDate: this.$moment(this.startDate).toISOString(true),
                        endDate: this.$moment(this.endDate).toISOString(true)
                    }, true))
                    .then(() => {
                        this.statsReady = true;
                    });
            }
        },
    };
</script>
<style lang="scss" scoped>
    @import "@kestra-io/ui-libs/src/scss/variables";

    .log-panel {
        > div.log-content {
            margin-bottom: var(--spacer);
            .navbar {
                border: 1px solid var(--bs-border-color);
            }
        }

        .logs-wrapper {
            margin-bottom: var(--spacer);
            border-radius: var(--bs-border-radius-lg);
            overflow: hidden;
            padding: $spacer;
            padding-top: calc($spacer/2);
            background-color: var(--bs-white);
            border: 1px solid var(--bs-border-color);

            html.dark & {
                background-color: var(--bs-gray-100);
            }

            > * + * {
                border-top: 1px solid var(--bs-border-color);
            }
        }
    }
</style>