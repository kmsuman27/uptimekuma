<template>
    <transition ref="tableContainer" name="slide-fade" appear>
        <div v-if="$route.name === 'DashboardHome'" class="monitor-hub">

            <!-- Header -->
            <div class="hub-header">
                <h1>Monitor Hub</h1>
                <p>Manage all your monitored services in one place</p>
            </div>

            <!-- Action Bar -->
            <div class="hub-action-bar">

                <div class="hub-actions">

                    <!-- Add Monitor -->
                    <router-link
                        to="/add"
                        class="hub-btn hub-btn-primary"
                    >
                        <font-awesome-icon icon="plus" />
                        <span>Add Monitor</span>
                    </router-link>

                    <!-- Bulk Import -->
                    <button
    class="hub-btn"
    type="button"
    @click="showBulkImportModal = true"
>
    <font-awesome-icon icon="upload" />
    Bulk Import
</button>

                    <!-- CSV Upload -->
                    <button
                        type="button"
                        class="hub-btn"
                        @click="showCSVImportModal = true"
                    >
                        <span class="button-icon">↑</span>
                        <span>CSV Upload</span>
                    </button>

                    <!-- Templates -->
                    <button
                        type="button"
                        class="hub-btn"
                    >
                        <span class="button-icon">▦</span>
                        <span>Templates</span>
                    </button>

                </div>

                <!-- Monitor Count -->
                <div class="monitor-count">
                    <strong>{{ monitorCount }}</strong>
                    <span>monitors</span>
                </div>

            </div>

            <!-- Empty State -->
            <div
                v-if="monitorCount === 0"
                class="empty-state"
            >

                <div class="empty-icon">
                    <span>◎</span>
                </div>

                <h2>No monitors monitored</h2>

                <p>
                    Add your first monitor to start monitoring uptime and availability
                </p>

                <div class="empty-actions">

                    <router-link
                        to="/add"
                        class="hub-btn hub-btn-primary"
                    >
                        <font-awesome-icon icon="plus" />
                        <span>Add Monitor</span>
                    </router-link>

                    <button
                        type="button"
                        class="hub-btn"
                    >
                        <span class="button-icon">☰</span>
                        <span>Bulk Import</span>
                    </button>

                </div>

            </div>

            <!-- Existing Monitor Dashboard -->
            <div
                v-else
                class="monitor-content"
            >

                <!-- Statistics -->
                <div class="stats-box">

                    <div class="stat-item">
                        <h3>{{ $t("Up") }}</h3>

                        <span
                            class="num"
                            :class="$root.stats.up === 0 && 'text-secondary'"
                        >
                            {{ $root.stats.up }}
                        </span>
                    </div>

                    <div class="stat-item">
                        <h3>{{ $t("Down") }}</h3>

                        <span
                            class="num"
                            :class="$root.stats.down > 0 ? 'text-danger' : 'text-secondary'"
                        >
                            {{ $root.stats.down }}
                        </span>
                    </div>

                    <div class="stat-item">
                        <h3>{{ $t("Maintenance") }}</h3>

                        <span
                            class="num"
                            :class="$root.stats.maintenance > 0 ? 'text-maintenance' : 'text-secondary'"
                        >
                            {{ $root.stats.maintenance }}
                        </span>
                    </div>

                    <div class="stat-item">
                        <h3>{{ $t("Unknown") }}</h3>

                        <span class="num text-secondary">
                            {{ $root.stats.unknown }}
                        </span>
                    </div>

                    <div class="stat-item">
                        <h3>{{ $t("pauseDashboardHome") }}</h3>

                        <span class="num text-secondary">
                            {{ $root.stats.pause }}
                        </span>
                    </div>

                </div>

                <!-- Events -->
                <div class="shadow-box table-shadow-box table-wrapper">

                    <div class="events-header">
                        <h2>Monitor Events</h2>

                        <button
                            class="btn btn-sm btn-outline-danger"
                            :disabled="clearingAllEvents"
                            @click="clearAllEventsDialog"
                        >
                            {{ $t("Clear All Events") }}
                        </button>
                    </div>

                    <table class="table table-borderless table-hover">

                        <thead>
                            <tr>

                                <th v-if="showGroupColumn">
                                    {{ $t("Group Name") }}
                                </th>

                                <th class="name-column">
                                    {{ $t("Name") }}
                                </th>

                                <th>
                                    {{ $t("Status") }}
                                </th>

                                <th>
                                    {{ $t("DateTime") }}
                                </th>

                                <th>
                                    {{ $t("Message") }}
                                </th>

                            </tr>
                        </thead>

                        <tbody>

                            <tr
                                v-for="(beat, index) in displayedRecords"
                                :key="index"
                                :class="{
                                    'shadow-box': $root.windowWidth <= 550
                                }"
                            >

                                <td v-if="showGroupColumn">

                                    <router-link
                                        v-if="getGroupName(beat.monitorID)"
                                        :to="`/dashboard/${getGroupId(beat.monitorID)}`"
                                    >
                                        {{ getGroupName(beat.monitorID) }}
                                    </router-link>

                                    <span
                                        v-else
                                        class="text-secondary"
                                    >
                                        —
                                    </span>

                                </td>

                                <td class="name-column">

                                    <router-link
                                        :to="`/dashboard/${beat.monitorID}`"
                                    >
                                        {{ $root.monitorList[beat.monitorID]?.name }}
                                    </router-link>

                                </td>

                                <td>
                                    <Status :status="beat.status" />
                                </td>

                                <td
                                    :class="{
                                        'border-0': !beat.msg
                                    }"
                                >
                                    <Datetime :value="beat.time" />
                                </td>

                                <td class="border-0">
                                    {{ beat.msg }}
                                </td>

                            </tr>

                            <tr
                                v-if="importantHeartBeatListLength === 0"
                            >
                                <td :colspan="tableColumnCount">
                                    {{ $t("No important events") }}
                                </td>
                            </tr>

                        </tbody>

                    </table>

                    <!-- Pagination -->
                    <div class="d-flex justify-content-center kuma_pagination">

                        <pagination
                            v-model="page"
                            :records="importantHeartBeatListLength"
                            :per-page="perPage"
                            :options="paginationConfig"
                        />

                    </div>

                </div>

            </div>

        </div>
    </transition>

    <!-- Clear Events Confirmation -->
    <Confirm
        ref="confirmClearEvents"
        btn-style="btn-danger"
        :yes-text="$t('Yes')"
        :no-text="$t('No')"
        @yes="clearAllEvents"
    >
        {{ $t("clearAllEventsMsg") }}
    </Confirm>

    <!-- Child Routes -->
    <router-view ref="child" />
    <BulkImportModal
    :show="showBulkImportModal"
    @close="showBulkImportModal = false"
    @imported="handleBulkImportResult"
/>
<CSVImportModal
    :show="showCSVImportModal"
    @close="showCSVImportModal = false"
    @imported="handleCSVImportResult"
/>
</template>


<script>
import Status from "../components/Status.vue";
import Datetime from "../components/Datetime.vue";
import Pagination from "v-pagination-3";
import Confirm from "../components/Confirm.vue";
import BulkImportModal from "../components/BulkImportModal.vue";
import CSVImportModal from "../components/CSVImportModal.vue";

export default {
    components: {
        Datetime,
        Status,
        Pagination,
        Confirm,
        BulkImportModal,
        CSVImportModal,
    },

    props: {
        calculatedHeight: {
            type: Number,
            default: 0,
        },
    },

    data() {
        return {
            page: 1,

            perPage: 25,

            initialPerPage: 25,

            paginationConfig: {
                hideCount: true,
                chunksNavigation: "scroll",
            },
            showBulkImportModal: false,
            showCSVImportModal: false,
            importantHeartBeatListLength: 0,
            

            displayedRecords: [],

            clearingAllEvents: false,
        };
    },

    computed: {

        /*
         * Total number of monitors
         */
        monitorCount() {
            return Object.keys(this.$root.monitorList).length;
        },

        /*
         * Check whether any monitor belongs to a group
         */
        showGroupColumn() {
            return Object.values(this.$root.monitorList).some(
                (m) => m.parent != null
            );
        },

        /*
         * Number of table columns
         */
        tableColumnCount() {
            return this.showGroupColumn ? 5 : 4;
        },
    },

    watch: {

        perPage() {
            this.$nextTick(() => {
                this.getImportantHeartbeatListPaged();
            });
        },

        page() {
            this.getImportantHeartbeatListPaged();
        },
    },

    mounted() {

        this.getImportantHeartbeatListLength();

        this.$root.emitter.on(
            "newImportantHeartbeat",
            this.onNewImportantHeartbeat
        );

        this.initialPerPage = this.perPage;

        window.addEventListener(
            "resize",
            this.updatePerPage
        );

        this.updatePerPage();
    },

    beforeUnmount() {

        this.$root.emitter.off(
            "newImportantHeartbeat",
            this.onNewImportantHeartbeat
        );

        window.removeEventListener(
            "resize",
            this.updatePerPage
        );
    },

    methods: {

        /*
         * Get monitor group name
         */
        getGroupName(monitorID) {

            const monitor =
                this.$root.monitorList[monitorID];

            if (!monitor || monitor.parent == null) {
                return "";
            }

            const parent =
                this.$root.monitorList[monitor.parent];

            return parent ? parent.name : "";
        },

        /*
         * Get monitor group ID
         */
        getGroupId(monitorID) {

            const monitor =
                this.$root.monitorList[monitorID];

            return monitor && monitor.parent != null
                ? monitor.parent
                : null;
        },

        /*
         * Handle new heartbeat
         */
        onNewImportantHeartbeat(heartbeat) {

            if (this.page === 1) {

                this.displayedRecords.unshift(
                    heartbeat
                );

                if (
                    this.displayedRecords.length >
                    this.perPage
                ) {
                    this.displayedRecords.pop();
                }

                this.importantHeartBeatListLength += 1;
            }
        },

        /*
         * Get total important heartbeat count
         */
        getImportantHeartbeatListLength() {

            this.$root
                .getSocket()
                .emit(
                    "monitorImportantHeartbeatListCount",
                    null,
                    (res) => {

                        if (res.ok) {

                            this.importantHeartBeatListLength =
                                res.count;

                            this.getImportantHeartbeatListPaged();
                        }
                    }
                );
        },

        /*
         * Get paginated heartbeat records
         */
        getImportantHeartbeatListPaged() {

            const offset =
                (this.page - 1) * this.perPage;

            this.$root
                .getSocket()
                .emit(
                    "monitorImportantHeartbeatListPaged",
                    null,
                    offset,
                    this.perPage,
                    (res) => {

                        if (res.ok) {
                            this.displayedRecords =
                                res.data;
                        }
                    }
                );
        },

        /*
         * Update records per page
         */
        updatePerPage() {

            const tableContainer =
                this.$refs.tableContainer;

            if (!tableContainer) {
                return;
            }

            const tableContainerHeight =
                tableContainer.offsetHeight;

            const availableHeight =
                window.innerHeight -
                tableContainerHeight;

            const additionalPerPage =
                Math.floor(
                    availableHeight / 58
                );

            if (additionalPerPage > 0) {

                this.perPage =
                    Math.max(
                        this.initialPerPage,
                        this.perPage +
                        additionalPerPage
                    );

            } else {

                this.perPage =
                    this.initialPerPage;
            }
        },

        /*
         * Open clear events dialog
         */
        clearAllEventsDialog() {
            this.$refs.confirmClearEvents.show();
        },

        /*
         * Clear events
         */
        clearAllEvents() {

            this.clearingAllEvents = true;

            const monitorIDs =
                Object.keys(
                    this.$root.monitorList
                );

            let failed = 0;

            const total =
                monitorIDs.length;

            if (total === 0) {

                this.clearingAllEvents = false;

                this.$root.toastError(
                    this.$t("No monitors found")
                );

                return;
            }

            monitorIDs.forEach(
                (monitorID) => {

                    this.$root
                        .getSocket()
                        .emit(
                            "clearEvents",
                            monitorID,
                            (res) => {

                                if (
                                    !res ||
                                    !res.ok
                                ) {
                                    failed++;
                                }
                            }
                        );
                }
            );

            this.clearingAllEvents = false;

            this.page = 1;

            this.getImportantHeartbeatListLength();

            if (failed === 0) {

                this.$root.toastSuccess(
                    this.$t(
                        "Events cleared successfully"
                    )
                );

            } else {

                             this.$root.toastError(
                    this.$t(
                        "Could not clear events",
                        {
                            failed,
                            total,
                        }
                    )
                );
            }
        },

       handleBulkImportResult(result) {
    this.showBulkImportModal = false;

    if (result.successCount > 0) {
        this.$root.toastSuccess(
            `${result.successCount} monitor(s) imported successfully`
        );
    }

    if (result.failedCount > 0) {
        this.$root.toastError(
            `${result.failedCount} monitor(s) failed to import`
        );
    }
},

handleCSVImportResult(result) {
    this.showCSVImportModal = false;

    if (result.successCount > 0) {
        this.$root.toastSuccess(
            `${result.successCount} monitor(s) imported successfully`
        );
    }

    if (result.failedCount > 0) {
        this.$root.toastError(
            `${result.failedCount} monitor(s) failed to import`
        );
    }
},
},
};
</script>
<style lang="scss" scoped>
@import "../assets/vars";


/* =========================================================
   MAIN CONTAINER
   ========================================================= */

.monitor-hub {
    width: 100%;
    padding: 10px 0 30px;
}


/* =========================================================
   HEADER
   ========================================================= */

.hub-header {
    margin-bottom: 22px;

    h1 {
        margin: 0 0 5px;

        font-size: 24px;
        font-weight: 600;
    }

    p {
        margin: 0;

        color: #6c757d;

        font-size: 14px;
    }
}


/* =========================================================
   ACTION BAR
   ========================================================= */

.hub-action-bar {
    display: flex;
    align-items: center;
    justify-content: space-between;

    min-height: 76px;

    padding: 16px 20px;

    margin-bottom: 20px;

    background: #ffffff;

    border: 1px solid #dee2e6;

    border-radius: 12px;
}


.hub-actions {
    display: flex;
    align-items: center;

    gap: 10px;
}


/* =========================================================
   BUTTON
   ========================================================= */

.hub-btn {
    display: inline-flex;

    align-items: center;
    justify-content: center;

    gap: 8px;

    min-height: 40px;

    padding: 9px 16px;

    background: #ffffff;

    border: 1px solid #ced4da;

    border-radius: 8px;

    color: #263238;

    font-size: 14px;

    font-weight: 500;

    text-decoration: none;

    cursor: pointer;

    transition:
        background-color 0.15s ease,
        border-color 0.15s ease,
        opacity 0.15s ease;
}


.hub-btn:hover {
    background: #f8f9fa;

    border-color: #adb5bd;
}


.hub-btn-primary {
    background: $primary;

    border-color: $primary;

    color: #ffffff;
}


.hub-btn-primary:hover {
    background: $primary;

    border-color: $primary;

    color: #ffffff;

    opacity: 0.9;
}


.button-icon {
    display: inline-flex;

    align-items: center;
    justify-content: center;

    width: 16px;

    font-size: 14px;

    line-height: 1;
}


/* =========================================================
   MONITOR COUNT
   ========================================================= */

.monitor-count {
    display: flex;

    align-items: center;

    gap: 5px;

    padding: 11px 16px;

    border-radius: 22px;

    background: rgba($primary, 0.1);

    color: $primary;

    font-size: 14px;
}


.monitor-count strong {
    font-size: 17px;
}


/* =========================================================
   EMPTY STATE
   ========================================================= */

.empty-state {
    min-height: 300px;

    display: flex;

    flex-direction: column;

    align-items: center;

    justify-content: center;

    padding: 50px 20px;

    background: #ffffff;

    border: 2px dashed #dee2e6;

    border-radius: 12px;

    text-align: center;
}


.empty-icon {
    width: 50px;

    height: 50px;

    display: flex;

    align-items: center;

    justify-content: center;

    margin-bottom: 16px;

    border-radius: 50%;

    background: #f1f3f5;

    color: #adb5bd;

    font-size: 28px;
}


.empty-state h2 {
    margin: 0 0 8px;

    font-size: 17px;

    font-weight: 500;
}


.empty-state p {
    margin: 0 0 20px;

    color: #6c757d;

    font-size: 14px;
}


/* =========================================================
   EMPTY ACTIONS
   ========================================================= */

.empty-actions {
    display: flex;

    align-items: center;

    gap: 10px;
}


/* =========================================================
   MONITOR CONTENT
   ========================================================= */

.monitor-content {
    margin-top: 10px;
}


/* =========================================================
   STATS
   ========================================================= */

.stats-box {
    display: grid;

    grid-template-columns:
        repeat(5, 1fr);

    padding: 20px;

    margin-bottom: 20px;

    background: #ffffff;

    border: 1px solid #dee2e6;

    border-radius: 12px;
}


.stat-item {
    text-align: center;
}


.stat-item h3 {
    margin-bottom: 5px;

    font-size: 14px;

    font-weight: 500;
}


.num {
    display: block;

    font-size: 28px;

    font-weight: 700;

    color: $primary;
}


/* =========================================================
   EVENTS
   ========================================================= */

.events-header {
    display: flex;

    align-items: center;

    justify-content: space-between;

    margin-bottom: 15px;
}


.events-header h2 {
    margin: 0;

    font-size: 18px;

    font-weight: 600;
}


/* =========================================================
   TABLE
   ========================================================= */

.shadow-box {
    padding: 20px;
}


.table-wrapper {
    overflow-x: auto;
}


table {
    font-size: 14px;
}


table tr {
    transition:
        all ease-in-out 0.2s;
}


@media (max-width: 550px) {

    table {
        table-layout: fixed;

        overflow-wrap: break-word;
    }
}


/* =========================================================
   RESPONSIVE
   ========================================================= */

@media screen and (max-width: 900px) {

    .hub-action-bar {
        align-items: flex-start;

        flex-direction: column;

        gap: 15px;
    }


    .hub-actions {
        flex-wrap: wrap;
    }


    .stats-box {
        grid-template-columns:
            repeat(2, 1fr);

        gap: 20px;
    }
}


@media screen and (max-width: 550px) {

    .hub-actions {
        width: 100%;
    }


    .hub-btn {
        flex: 1;
    }


    .empty-actions {
        flex-direction: column;

        width: 100%;
    }


    .empty-actions .hub-btn {
        width: 100%;
    }


    .stats-box {
        grid-template-columns: 1fr;
    }


    .monitor-count {
        align-self: flex-end;
    }
}


@media screen and (max-width: 1280px) {

    .name-column {
        min-width: 150px;
    }
}


@media screen and (min-aspect-ratio: 4/3) {

    .name-column {
        min-width: 200px;
    }
}
</style>