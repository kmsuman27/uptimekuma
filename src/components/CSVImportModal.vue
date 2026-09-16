<template>
    <div
        v-if="show"
        class="modal fade show d-block"
        tabindex="-1"
        role="dialog"
        aria-modal="true"
    >
        <div class="modal-dialog modal-xl modal-dialog-centered">
            <div class="modal-content">
                <div class="modal-header">
                    <div>
                        <h5 class="modal-title">CSV Upload</h5>
                        <p class="modal-subtitle mb-0">
                            Upload a CSV file to import multiple HTTP monitors
                        </p>
                    </div>

                    <button
                        type="button"
                        class="btn-close"
                        aria-label="Close"
                        :disabled="importing"
                        @click="close"
                    ></button>
                </div>

                <div class="modal-body">
                    <div class="upload-box">
                        <label class="form-label">
                            Select CSV file
                        </label>

                        <input
                            ref="fileInput"
                            type="file"
                            class="form-control"
                            accept=".csv,text/csv"
                            :disabled="importing"
                            @change="handleFile"
                        />

                        <div class="form-text">
                            Required columns:
                            <strong>name, url</strong>.
                            Optional column:
                            <strong>interval</strong>.
                        </div>
                    </div>

                    <div v-if="fileName" class="selected-file mt-3">
                        Selected file:
                        <strong>{{ fileName }}</strong>
                    </div>

                    <div
                        v-if="parseError"
                        class="alert alert-danger mt-3"
                    >
                        {{ parseError }}
                    </div>

                    <div
                        v-if="rows.length"
                        class="preview-section mt-3"
                    >
                        <div class="preview-header">
                            <strong>
                                Preview ({{ rows.length }})
                            </strong>
                        </div>

                        <div class="table-responsive">
                            <table class="table table-sm">
                                <thead>
                                    <tr>
                                        <th>#</th>
                                        <th>Name</th>
                                        <th>URL</th>
                                        <th>Interval</th>
                                        <th>Status</th>
                                    </tr>
                                </thead>

                                <tbody>
                                    <tr
                                        v-for="(row, index) in rows"
                                        :key="index"
                                    >
                                        <td>{{ index + 1 }}</td>

                                        <td>
                                            {{ row.name }}
                                        </td>

                                        <td>
                                            {{ row.url }}
                                        </td>

                                        <td>
                                            {{ row.interval }}
                                        </td>

                                        <td>
                                            <span
                                                v-if="row.valid"
                                                class="badge bg-success"
                                            >
                                                Valid
                                            </span>

                                            <span
                                                v-else
                                                class="badge bg-danger"
                                            >
                                                {{ row.error }}
                                            </span>
                                        </td>
                                    </tr>
                                </tbody>
                            </table>
                        </div>
                    </div>

                    <div
                        v-if="importing"
                        class="import-progress mt-3"
                    >
                        Importing
                        {{ currentIndex }}
                        /
                        {{ validRows.length }}
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
                        :disabled="
                            importing ||
                            validRows.length === 0
                        "
                        @click="importRows"
                    >
                        <span
                            v-if="importing"
                            class="spinner-border spinner-border-sm me-2"
                        ></span>

                        {{
                            importing
                                ? "Importing..."
                                : "Import Monitors"
                        }}
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
};

export default {
    name: "CSVImportModal",

    props: {
        show: {
            type: Boolean,
            default: false,
        },
    },

    emits: ["close", "imported"],

    data() {
        return {
            fileName: "",
            rows: [],
            parseError: "",
            importing: false,
            currentIndex: 0,
        };
    },

    computed: {
        validRows() {
            return this.rows.filter((row) => row.valid);
        },
    },

    methods: {
        close() {
            if (this.importing) {
                return;
            }

            this.reset();
            this.$emit("close");
        },

        reset() {
            this.fileName = "";
            this.rows = [];
            this.parseError = "";
            this.currentIndex = 0;

            if (this.$refs.fileInput) {
                this.$refs.fileInput.value = "";
            }
        },

        parseCSVLine(line) {
            const values = [];
            let current = "";
            let insideQuotes = false;

            for (let i = 0; i < line.length; i++) {
                const char = line[i];

                if (char === '"') {
                    if (
                        insideQuotes &&
                        line[i + 1] === '"'
                    ) {
                        current += '"';
                        i++;
                    } else {
                        insideQuotes = !insideQuotes;
                    }
                } else if (
                    char === "," &&
                    !insideQuotes
                ) {
                    values.push(current.trim());
                    current = "";
                } else {
                    current += char;
                }
            }

            values.push(current.trim());

            return values;
        },

        parseCSV(text) {
            const lines = text
                .replace(/^\uFEFF/, "")
                .split(/\r?\n/)
                .map((line) => line.trim())
                .filter(Boolean);

            if (lines.length < 2) {
                throw new Error(
                    "CSV must contain a header and at least one data row."
                );
            }

            const headers = this.parseCSVLine(
                lines[0]
            ).map((header) =>
                header.toLowerCase().trim()
            );

            const nameIndex = headers.indexOf("name");
            const urlIndex = headers.indexOf("url");
            const intervalIndex =
                headers.indexOf("interval");

            if (
                nameIndex === -1 ||
                urlIndex === -1
            ) {
                throw new Error(
                    "CSV must contain name and url columns."
                );
            }

            return lines.slice(1).map((line) => {
                const values =
                    this.parseCSVLine(line);

                const name =
                    values[nameIndex] || "";

                const url =
                    values[urlIndex] || "";

                const intervalValue =
                    intervalIndex !== -1
                        ? values[intervalIndex]
                        : "60";

                const interval =
                    Number(intervalValue) || 60;

                if (!name.trim()) {
                    return {
                        name,
                        url,
                        interval,
                        valid: false,
                        error: "Name is required",
                    };
                }

                if (!url.trim()) {
                    return {
                        name,
                        url,
                        interval,
                        valid: false,
                        error: "URL is required",
                    };
                }

                try {
                    const parsedUrl =
                        new URL(url);

                    if (
                        ![
                            "http:",
                            "https:",
                        ].includes(
                            parsedUrl.protocol
                        )
                    ) {
                        throw new Error();
                    }
                } catch (e) {
                    return {
                        name,
                        url,
                        interval,
                        valid: false,
                        error:
                            "Invalid HTTP/HTTPS URL",
                    };
                }

                if (interval < 20) {
                    return {
                        name,
                        url,
                        interval,
                        valid: false,
                        error:
                            "Interval must be at least 20 seconds",
                    };
                }

                return {
                    name: name.trim(),
                    url: url.trim(),
                    interval,
                    valid: true,
                };
            });
        },

        handleFile(event) {
            const file =
                event.target.files[0];

            if (!file) {
                return;
            }

            this.fileName = file.name;
            this.rows = [];
            this.parseError = "";

            const reader = new FileReader();

            reader.onload = () => {
                try {
                    this.rows =
                        this.parseCSV(
                            reader.result
                        );
                } catch (error) {
                    this.parseError =
                        error.message;
                }
            };

            reader.onerror = () => {
                this.parseError =
                    "Could not read the selected CSV file.";
            };

            reader.readAsText(file);
        },

        createMonitor(row) {
            return new Promise((resolve) => {
                const monitor = {
                    ...monitorDefaults,
                    name: row.name,
                    url: row.url,
                    interval: row.interval,
                    humanReadableInterval:
                        `${row.interval} seconds`,
                    type: "http",
                    notificationIDList: {},
                };

                this.$root.add(
                    monitor,
                    (res) => {
                        resolve(res);
                    }
                );
            });
        },

        async importRows() {
            if (
                this.validRows.length === 0
            ) {
                return;
            }

            this.importing = true;
            this.currentIndex = 0;

            let successCount = 0;
            let failedCount = 0;

            for (const row of this.validRows) {
                this.currentIndex++;

                try {
                    const res =
                        await this.createMonitor(
                            row
                        );

                    if (res && res.ok) {
                        successCount++;
                    } else {
                        failedCount++;
                    }
                } catch (error) {
                    failedCount++;
                }
            }

            this.importing = false;

            this.$emit("imported", {
                successCount,
                failedCount,
            });

            this.reset();
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

.upload-box {
    padding: 16px;
    border: 1px dashed #ced4da;
    border-radius: 8px;
}

.selected-file {
    font-size: 14px;
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