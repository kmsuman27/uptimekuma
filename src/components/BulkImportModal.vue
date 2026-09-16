<template>
    <div
        v-if="show"
        class="modal fade show d-block"
        tabindex="-1"
        role="dialog"
        aria-modal="true"
    >
        <div class="modal-dialog modal-lg modal-dialog-centered">
            <div class="modal-content">
                <div class="modal-header">
                    <div>
                        <h5 class="modal-title">Bulk Import Monitors</h5>
                        <p class="modal-subtitle mb-0">
                            Add multiple HTTP monitors at once
                        </p>
                    </div>

                    <button
                        type="button"
                        class="btn-close"
                        aria-label="Close"
                        @click="close"
                    ></button>
                </div>

                <div class="modal-body">
                    <div class="form-group mb-3">
                        <label class="form-label">
                            Monitors
                        </label>

                        <textarea
                            v-model="input"
                            class="form-control"
                            rows="10"
                            placeholder="Google | https://google.com
GitHub | https://github.com
My Website | https://example.com"
                        ></textarea>

                        <div class="form-text">
                            One monitor per line. Format:
                            <strong>Name | URL</strong>
                        </div>
                    </div>

                    <div v-if="parsedMonitors.length" class="preview-section">
                        <div class="preview-header">
                            <strong>
                                Preview ({{ parsedMonitors.length }})
                            </strong>
                        </div>

                        <div class="table-responsive">
                            <table class="table table-sm">
                                <thead>
                                    <tr>
                                        <th>#</th>
                                        <th>Name</th>
                                        <th>URL</th>
                                        <th>Status</th>
                                    </tr>
                                </thead>

                                <tbody>
                                    <tr
                                        v-for="(monitor, index) in parsedMonitors"
                                        :key="index"
                                    >
                                        <td>{{ index + 1 }}</td>
                                        <td>{{ monitor.name }}</td>
                                        <td>{{ monitor.url }}</td>
                                        <td>
                                            <span
                                                v-if="monitor.valid"
                                                class="badge bg-success"
                                            >
                                                Valid
                                            </span>

                                            <span
                                                v-else
                                                class="badge bg-danger"
                                            >
                                                {{ monitor.error }}
                                            </span>
                                        </td>
                                    </tr>
                                </tbody>
                            </table>
                        </div>
                    </div>

                    <div v-if="importing" class="import-progress mt-3">
                        Importing {{ currentIndex }} / {{ validMonitors.length }}
                    </div>
                </div>

                <div class="modal-footer">
                    <button
                        type="button"
                        class="btn btn-secondary"
                        :disabled="importing"
                        @click="close"
                    >
                        Cancel
                    </button>

                    <button
                        type="button"
                        class="btn btn-primary"
                        :disabled="importing || validMonitors.length === 0"
                        @click="importMonitors"
                    >
                        <span
                            v-if="importing"
                            class="spinner-border spinner-border-sm me-2"
                        ></span>

                        {{ importing ? "Importing..." : "Import Monitors" }}
                    </button>
                </div>
            </div>
        </div>
    </div>

    <div
        v-if="show"
        class="modal-backdrop fade show"
        @click="!importing && close"
    ></div>
</template>

<script>
const monitorDefaults = {
    type: "http",
    name: "",
    parent: null,
    url: "https://",
    wsSubprotocol: "",
    method: "GET",
    protocol: null,
    location: "world",
    ipFamily: null,
    interval: 60,
    humanReadableInterval: "60 seconds",
    retryInterval: 60,
    resendInterval: 0,
    maxretries: 0,
    retryOnlyOnStatusCodeFailure: false,
    notificationIDList: {},
    ignoreTls: false,
    upsideDown: false,
    expiryNotification: false,
    domainExpiryNotification: true,
    maxredirects: 10,
    accepted_statuscodes: ["200-299"],
    saveResponse: false,
    saveErrorResponse: true,
    responseMaxLength: 1024,
    dns_resolve_type: "A",
    dns_resolve_server: "",
    docker_container: "",
    docker_host: null,
    proxyId: null,
    basic_auth_user: "",
    basic_auth_pass: "",
    bearer_token: "",
    mqttUsername: "",
    mqttPassword: "",
    mqttTopic: "",
    mqttWebsocketPath: "",
    mqttSuccessMessage: "",
    mqttCheckType: "keyword",
    authMethod: null,
    oauth_auth_method: "client_secret_basic",
    httpBodyEncoding: "json",
    kafkaProducerBrokers: [],
    kafkaProducerSaslOptions: {
        mechanism: "None",
    },
    cacheBust: false,
    kafkaProducerSsl: false,
    kafkaProducerAllowAutoTopicCreation: false,
    gamedigGivenPortOnly: true,
    gamedigToken: "",
    remote_browser: null,
    screenshot_delay: 0,
    rabbitmqNodes: [],
    rabbitmqUsername: "",
    rabbitmqPassword: "",
    conditions: [],
    system_service_name: "",
    sshAuthMethod: "password",
    ntpStratumThreshold: 5,
    ntpTimeOffsetThreshold: 1000,
    ntpRootDispersionThreshold: 500,
};

export default {
    name: "BulkImportModal",

    props: {
        show: {
            type: Boolean,
            default: false,
        },
    },

    emits: ["close", "imported"],

    data() {
        return {
            input: "",
            importing: false,
            currentIndex: 0,
        };
    },

    computed: {
        parsedMonitors() {
            if (!this.input.trim()) {
                return [];
            }

            return this.input
                .split(/\r?\n/)
                .map((line) => line.trim())
                .filter(Boolean)
                .map((line) => {
                    const separatorIndex = line.indexOf("|");

                    if (separatorIndex === -1) {
                        return {
                            name: line,
                            url: "",
                            valid: false,
                            error: "Missing | separator",
                        };
                    }

                    const name = line.slice(0, separatorIndex).trim();
                    const url = line.slice(separatorIndex + 1).trim();

                    if (!name) {
                        return {
                            name: "",
                            url,
                            valid: false,
                            error: "Name is required",
                        };
                    }

                    if (!url) {
                        return {
                            name,
                            url: "",
                            valid: false,
                            error: "URL is required",
                        };
                    }

                    try {
                        const parsedUrl = new URL(url);

                        if (!["http:", "https:"].includes(parsedUrl.protocol)) {
                            throw new Error("Only HTTP/HTTPS URLs are supported");
                        }

                        return {
                            name,
                            url,
                            valid: true,
                        };
                    } catch (e) {
                        return {
                            name,
                            url,
                            valid: false,
                            error: "Invalid HTTP/HTTPS URL",
                        };
                    }
                });
        },

        validMonitors() {
            return this.parsedMonitors.filter((monitor) => monitor.valid);
        },
    },

    methods: {
        close() {
            if (this.importing) {
                return;
            }

            this.$emit("close");
        },

        createMonitor(data) {
            return new Promise((resolve) => {
                const monitor = {
                    ...monitorDefaults,
                    name: data.name,
                    url: data.url,
                    type: "http",
                    notificationIDList: {},
                };

                this.$root.add(monitor, (res) => {
                    resolve(res);
                });
            });
        },

        async importMonitors() {
            if (this.validMonitors.length === 0) {
                return;
            }

            this.importing = true;
            this.currentIndex = 0;

            let successCount = 0;
            let failedCount = 0;

            for (const monitor of this.validMonitors) {
                this.currentIndex++;

                try {
                    const res = await this.createMonitor(monitor);

                    if (res && res.ok) {
                        successCount++;
                    } else {
                        failedCount++;
                    }
                } catch (e) {
                    failedCount++;
                }
            }

            this.importing = false;

            this.$emit("imported", {
                successCount,
                failedCount,
            });

            this.input = "";
            this.currentIndex = 0;
        },
    },
};
</script>

<style scoped>
.modal {
    z-index: 1055;
}

.modal-backdrop {
    z-index: 1050;
}

.modal-subtitle {
    color: #6c757d;
    font-size: 13px;
}

.preview-section {
    border: 1px solid #dee2e6;
    border-radius: 8px;
    overflow: hidden;
}

.preview-header {
    padding: 10px 14px;
    background: #f8f9fa;
    border-bottom: 1px solid #dee2e6;
}

.table {
    margin-bottom: 0;
}

.import-progress {
    font-size: 14px;
    font-weight: 500;
}
</style>