{% code title="jobs.json" %}

```json
{
    "annotations": {
        "list": [
            {
                "builtIn": 1,
                "datasource": {
                    "type": "datasource",
                    "uid": "grafana"
                },
                "enable": true,
                "hide": true,
                "iconColor": "rgba(0, 211, 255, 1)",
                "name": "Annotations & Alerts",
                "target": {
                    "limit": 100,
                    "matchAny": false,
                    "tags": [],
                    "type": "dashboard"
                },
                "type": "dashboard"
            }
        ]
    },
    "editable": true,
    "fiscalYearStartMonth": 0,
    "graphTooltip": 0,
    "id": 1,
    "links": [
        {
            "asDropdown": false,
            "icon": "external link",
            "includeVars": false,
            "keepTime": false,
            "tags": [],
            "targetBlank": false,
            "title": "New link",
            "tooltip": "",
            "type": "dashboards",
            "url": ""
        }
    ],
    "liveNow": false,
    "panels": [
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "thresholds"
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "green",
                                "value": null
                            }
                        ]
                    },
                    "unit": "s"
                },
                "overrides": []
            },
            "gridPos": {
                "h": 9,
                "w": 3,
                "x": 0,
                "y": 0
            },
            "id": 20,
            "options": {
                "colorMode": "value",
                "graphMode": "area",
                "justifyMode": "auto",
                "orientation": "auto",
                "reduceOptions": {
                    "calcs": [
                        "last"
                    ],
                    "fields": "",
                    "values": false
                },
                "textMode": "auto"
            },
            "pluginVersion": "9.4.1",
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": false,
                    "expr": "rr_http_uptime_seconds",
                    "instant": true,
                    "interval": "",
                    "legendFormat": "Uptime",
                    "refId": "A"
                }
            ],
            "title": "RR uptime",
            "type": "stat"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "palette-classic"
                    },
                    "custom": {
                        "axisCenteredZero": false,
                        "axisColorMode": "text",
                        "axisLabel": "",
                        "axisPlacement": "auto",
                        "barAlignment": 0,
                        "drawStyle": "line",
                        "fillOpacity": 4,
                        "gradientMode": "none",
                        "hideFrom": {
                            "legend": false,
                            "tooltip": false,
                            "viz": false
                        },
                        "lineInterpolation": "linear",
                        "lineWidth": 1,
                        "pointSize": 5,
                        "scaleDistribution": {
                            "type": "linear"
                        },
                        "showPoints": "auto",
                        "spanNulls": 3600000,
                        "stacking": {
                            "group": "A",
                            "mode": "none"
                        },
                        "thresholdsStyle": {
                            "mode": "off"
                        }
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "yellow",
                                "value": null
                            }
                        ]
                    },
                    "unit": "decbytes"
                },
                "overrides": [
                    {
                        "matcher": {
                            "id": "byName",
                            "options": "RR memory"
                        },
                        "properties": [
                            {
                                "id": "color",
                                "value": {
                                    "fixedColor": "blue",
                                    "mode": "fixed"
                                }
                            }
                        ]
                    }
                ]
            },
            "gridPos": {
                "h": 9,
                "w": 10,
                "x": 3,
                "y": 0
            },
            "id": 14,
            "options": {
                "legend": {
                    "calcs": [],
                    "displayMode": "list",
                    "placement": "bottom",
                    "showLegend": true
                },
                "tooltip": {
                    "mode": "single",
                    "sort": "none"
                }
            },
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": true,
                    "expr": "go_memstats_heap_inuse_bytes",
                    "interval": "",
                    "legendFormat": "RR memory",
                    "refId": "A"
                }
            ],
            "title": "RR in-use memory",
            "type": "timeseries"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "palette-classic"
                    },
                    "custom": {
                        "axisCenteredZero": false,
                        "axisColorMode": "text",
                        "axisLabel": "",
                        "axisPlacement": "auto",
                        "barAlignment": 0,
                        "drawStyle": "line",
                        "fillOpacity": 11,
                        "gradientMode": "hue",
                        "hideFrom": {
                            "legend": false,
                            "tooltip": false,
                            "viz": false
                        },
                        "lineInterpolation": "linear",
                        "lineStyle": {
                            "fill": "solid"
                        },
                        "lineWidth": 1,
                        "pointSize": 5,
                        "scaleDistribution": {
                            "type": "linear"
                        },
                        "showPoints": "auto",
                        "spanNulls": 3600000,
                        "stacking": {
                            "group": "A",
                            "mode": "none"
                        },
                        "thresholdsStyle": {
                            "mode": "off"
                        }
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "green",
                                "value": null
                            },
                            {
                                "color": "red",
                                "value": 80
                            }
                        ]
                    }
                },
                "overrides": [
                    {
                        "matcher": {
                            "id": "byName",
                            "options": "Green threads count"
                        },
                        "properties": [
                            {
                                "id": "color",
                                "value": {
                                    "fixedColor": "purple",
                                    "mode": "fixed"
                                }
                            }
                        ]
                    }
                ]
            },
            "gridPos": {
                "h": 9,
                "w": 11,
                "x": 13,
                "y": 0
            },
            "id": 12,
            "options": {
                "legend": {
                    "calcs": [],
                    "displayMode": "list",
                    "placement": "bottom",
                    "showLegend": true
                },
                "tooltip": {
                    "mode": "single",
                    "sort": "none"
                }
            },
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": true,
                    "expr": "go_goroutines",
                    "instant": false,
                    "interval": "",
                    "legendFormat": "Green threads count",
                    "refId": "A"
                }
            ],
            "title": "Goroutines",
            "type": "timeseries"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "thresholds"
                    },
                    "mappings": [
                        {
                            "options": {
                                "pattern": "TOTAL",
                                "result": {
                                    "color": "red",
                                    "index": 0
                                }
                            },
                            "type": "regex"
                        }
                    ],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "blue",
                                "value": null
                            }
                        ]
                    }
                },
                "overrides": [
                    {
                        "matcher": {
                            "id": "byName",
                            "options": "TOTAL"
                        },
                        "properties": [
                            {
                                "id": "color",
                                "value": {
                                    "mode": "continuous-RdYlGr"
                                }
                            }
                        ]
                    },
                    {
                        "matcher": {
                            "id": "byName",
                            "options": "WORKING"
                        },
                        "properties": [
                            {
                                "id": "color",
                                "value": {
                                    "mode": "continuous-GrYlRd"
                                }
                            }
                        ]
                    },
                    {
                        "matcher": {
                            "id": "byName",
                            "options": "INVALID"
                        },
                        "properties": [
                            {
                                "id": "color",
                                "value": {
                                    "fixedColor": "dark-red",
                                    "mode": "fixed"
                                }
                            }
                        ]
                    }
                ]
            },
            "gridPos": {
                "h": 9,
                "w": 12,
                "x": 0,
                "y": 9
            },
            "id": 16,
            "options": {
                "displayMode": "lcd",
                "minVizHeight": 10,
                "minVizWidth": 0,
                "orientation": "horizontal",
                "reduceOptions": {
                    "calcs": [
                        "lastNotNull"
                    ],
                    "fields": "",
                    "values": false
                },
                "showUnfilled": true
            },
            "pluginVersion": "9.4.1",
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": false,
                    "expr": "rr_jobs_total_workers",
                    "hide": false,
                    "instant": true,
                    "interval": "",
                    "legendFormat": "TOTAL",
                    "refId": "C"
                },
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": false,
                    "expr": "rr_jobs_workers_ready",
                    "instant": true,
                    "interval": "",
                    "legendFormat": "READY",
                    "refId": "A"
                },
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": false,
                    "expr": "rr_jobs_workers_working",
                    "hide": false,
                    "instant": true,
                    "interval": "",
                    "legendFormat": "WORKING",
                    "refId": "D"
                },
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": false,
                    "expr": "rr_jobs_workers_invalid",
                    "hide": false,
                    "instant": true,
                    "interval": "",
                    "legendFormat": "INVALID",
                    "refId": "B"
                }
            ],
            "title": "Jobs workers state",
            "type": "bargauge"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "continuous-GrYlRd"
                    },
                    "custom": {
                        "axisCenteredZero": false,
                        "axisColorMode": "text",
                        "axisLabel": "",
                        "axisPlacement": "auto",
                        "barAlignment": 0,
                        "drawStyle": "line",
                        "fillOpacity": 5,
                        "gradientMode": "none",
                        "hideFrom": {
                            "legend": false,
                            "tooltip": false,
                            "viz": false
                        },
                        "lineInterpolation": "smooth",
                        "lineWidth": 2,
                        "pointSize": 5,
                        "scaleDistribution": {
                            "type": "linear"
                        },
                        "showPoints": "auto",
                        "spanNulls": 3600000,
                        "stacking": {
                            "group": "A",
                            "mode": "none"
                        },
                        "thresholdsStyle": {
                            "mode": "off"
                        }
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "green",
                                "value": null
                            },
                            {
                                "color": "red",
                                "value": 80
                            }
                        ]
                    },
                    "unit": "decmbytes"
                },
                "overrides": [
                    {
                        "matcher": {
                            "id": "byName",
                            "options": "Workers total memory used"
                        },
                        "properties": [
                            {
                                "id": "color",
                                "value": {
                                    "fixedColor": "green",
                                    "mode": "fixed"
                                }
                            }
                        ]
                    }
                ]
            },
            "gridPos": {
                "h": 9,
                "w": 12,
                "x": 12,
                "y": 9
            },
            "id": 6,
            "options": {
                "legend": {
                    "calcs": [],
                    "displayMode": "list",
                    "placement": "bottom",
                    "showLegend": true
                },
                "tooltip": {
                    "mode": "single",
                    "sort": "none"
                }
            },
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": true,
                    "expr": "(rr_jobs_workers_memory_bytes/1024000)",
                    "instant": false,
                    "interval": "",
                    "legendFormat": "Workers total memory used",
                    "refId": "A"
                }
            ],
            "title": "Workers total memory [JOBS]",
            "type": "timeseries"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "thresholds"
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "green",
                                "value": null
                            }
                        ]
                    },
                    "unit": "bytes"
                },
                "overrides": []
            },
            "gridPos": {
                "h": 12,
                "w": 24,
                "x": 0,
                "y": 18
            },
            "id": 10,
            "options": {
                "displayMode": "lcd",
                "minVizHeight": 10,
                "minVizWidth": 0,
                "orientation": "horizontal",
                "reduceOptions": {
                    "calcs": [
                        "lastNotNull"
                    ],
                    "fields": "",
                    "values": false
                },
                "showUnfilled": true
            },
            "pluginVersion": "9.4.1",
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": false,
                    "expr": "rr_jobs_worker_memory_bytes",
                    "instant": true,
                    "interval": "",
                    "legendFormat": "PID: {{pid}}",
                    "refId": "A"
                }
            ],
            "title": "Memory by worker [JOBS]",
            "type": "bargauge"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "palette-classic"
                    },
                    "custom": {
                        "axisCenteredZero": false,
                        "axisColorMode": "text",
                        "axisLabel": "",
                        "axisPlacement": "auto",
                        "barAlignment": 0,
                        "drawStyle": "line",
                        "fillOpacity": 0,
                        "gradientMode": "none",
                        "hideFrom": {
                            "legend": false,
                            "tooltip": false,
                            "viz": false
                        },
                        "lineInterpolation": "linear",
                        "lineWidth": 1,
                        "pointSize": 5,
                        "scaleDistribution": {
                            "type": "linear"
                        },
                        "showPoints": "auto",
                        "spanNulls": false,
                        "stacking": {
                            "group": "A",
                            "mode": "none"
                        },
                        "thresholdsStyle": {
                            "mode": "off"
                        }
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "green"
                            },
                            {
                                "color": "red",
                                "value": 80
                            }
                        ]
                    }
                },
                "overrides": []
            },
            "gridPos": {
                "h": 8,
                "w": 12,
                "x": 0,
                "y": 30
            },
            "id": 22,
            "options": {
                "legend": {
                    "calcs": [],
                    "displayMode": "list",
                    "placement": "bottom",
                    "showLegend": true
                },
                "tooltip": {
                    "mode": "single",
                    "sort": "none"
                }
            },
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": true,
                    "expr": "rate(rr_jobs_jobs_ok[5m])",
                    "interval": "",
                    "legendFormat": "",
                    "refId": "A"
                }
            ],
            "title": "JOBS Ok  rate [5m]",
            "type": "timeseries"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "palette-classic"
                    },
                    "custom": {
                        "axisCenteredZero": false,
                        "axisColorMode": "text",
                        "axisLabel": "",
                        "axisPlacement": "auto",
                        "barAlignment": 0,
                        "drawStyle": "line",
                        "fillOpacity": 9,
                        "gradientMode": "none",
                        "hideFrom": {
                            "legend": false,
                            "tooltip": false,
                            "viz": false
                        },
                        "lineInterpolation": "linear",
                        "lineWidth": 1,
                        "pointSize": 5,
                        "scaleDistribution": {
                            "type": "linear"
                        },
                        "showPoints": "auto",
                        "spanNulls": 3600000,
                        "stacking": {
                            "group": "A",
                            "mode": "none"
                        },
                        "thresholdsStyle": {
                            "mode": "off"
                        }
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "green"
                            }
                        ]
                    }
                },
                "overrides": [
                    {
                        "matcher": {
                            "id": "byName",
                            "options": "{instance=\"127.0.0.1:2112\", job=\"prometheus\"}"
                        },
                        "properties": [
                            {
                                "id": "color",
                                "value": {
                                    "fixedColor": "dark-red",
                                    "mode": "fixed"
                                }
                            }
                        ]
                    }
                ]
            },
            "gridPos": {
                "h": 8,
                "w": 12,
                "x": 12,
                "y": 30
            },
            "id": 26,
            "options": {
                "legend": {
                    "calcs": [],
                    "displayMode": "list",
                    "placement": "bottom",
                    "showLegend": true
                },
                "tooltip": {
                    "mode": "single",
                    "sort": "none"
                }
            },
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "exemplar": true,
                    "expr": "rate(rr_jobs_jobs_err[5m])",
                    "interval": "",
                    "legendFormat": "",
                    "refId": "A"
                }
            ],
            "title": "JOBS errors rate [5m]",
            "type": "timeseries"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "description": "",
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "palette-classic"
                    },
                    "custom": {
                        "axisCenteredZero": false,
                        "axisColorMode": "text",
                        "axisLabel": "",
                        "axisPlacement": "auto",
                        "barAlignment": 0,
                        "drawStyle": "line",
                        "fillOpacity": 0,
                        "gradientMode": "none",
                        "hideFrom": {
                            "legend": false,
                            "tooltip": false,
                            "viz": false
                        },
                        "lineInterpolation": "linear",
                        "lineWidth": 1,
                        "pointSize": 5,
                        "scaleDistribution": {
                            "type": "linear"
                        },
                        "showPoints": "auto",
                        "spanNulls": false,
                        "stacking": {
                            "group": "A",
                            "mode": "none"
                        },
                        "thresholdsStyle": {
                            "mode": "off"
                        }
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "green"
                            },
                            {
                                "color": "red",
                                "value": 10
                            }
                        ]
                    }
                },
                "overrides": []
            },
            "gridPos": {
                "h": 8,
                "w": 12,
                "x": 0,
                "y": 38
            },
            "id": 28,
            "options": {
                "legend": {
                    "calcs": [],
                    "displayMode": "list",
                    "placement": "bottom",
                    "showLegend": true
                },
                "tooltip": {
                    "mode": "single",
                    "sort": "none"
                }
            },
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "editorMode": "builder",
                    "exemplar": false,
                    "expr": "rr_jobs_push_latency_sum",
                    "instant": false,
                    "legendFormat": "{{driver}}-{{instance}}",
                    "range": true,
                    "refId": "A"
                }
            ],
            "title": "JOBS Push latency",
            "type": "timeseries"
        },
        {
            "datasource": {
                "type": "prometheus",
                "uid": "${DS_PROMETHEUS}"
            },
            "fieldConfig": {
                "defaults": {
                    "color": {
                        "mode": "palette-classic"
                    },
                    "custom": {
                        "axisCenteredZero": false,
                        "axisColorMode": "text",
                        "axisLabel": "",
                        "axisPlacement": "auto",
                        "barAlignment": 0,
                        "drawStyle": "line",
                        "fillOpacity": 0,
                        "gradientMode": "none",
                        "hideFrom": {
                            "legend": false,
                            "tooltip": false,
                            "viz": false
                        },
                        "lineInterpolation": "linear",
                        "lineWidth": 1,
                        "pointSize": 5,
                        "scaleDistribution": {
                            "type": "linear"
                        },
                        "showPoints": "auto",
                        "spanNulls": false,
                        "stacking": {
                            "group": "A",
                            "mode": "none"
                        },
                        "thresholdsStyle": {
                            "mode": "off"
                        }
                    },
                    "mappings": [],
                    "thresholds": {
                        "mode": "absolute",
                        "steps": [
                            {
                                "color": "green"
                            },
                            {
                                "color": "red",
                                "value": 80
                            }
                        ]
                    }
                },
                "overrides": []
            },
            "gridPos": {
                "h": 8,
                "w": 12,
                "x": 12,
                "y": 38
            },
            "id": 30,
            "options": {
                "legend": {
                    "calcs": [],
                    "displayMode": "list",
                    "placement": "bottom",
                    "showLegend": true
                },
                "tooltip": {
                    "mode": "single",
                    "sort": "none"
                }
            },
            "targets": [
                {
                    "datasource": {
                        "type": "prometheus",
                        "uid": "${DS_PROMETHEUS}"
                    },
                    "editorMode": "builder",
                    "expr": "rr_jobs_requests_total",
                    "legendFormat": "__auto",
                    "range": true,
                    "refId": "A"
                }
            ],
            "title": "JOBS total push requests",
            "type": "timeseries"
        }
    ],
    "refresh": "5s",
    "revision": 1,
    "schemaVersion": 38,
    "style": "dark",
    "tags": [],
    "templating": {
        "list": []
    },
    "time": {
        "from": "now-30m",
        "to": "now"
    },
    "timepicker": {},
    "timezone": "",
    "title": "RoadRunner JOBS dashboard",
    "uid": "fFGSls335lfs",
    "version": 5,
    "weekStart": "monday"
}

```

{% endcode %}
