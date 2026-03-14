idf_component_register(
    SRCS "main.cpp"
    INCLUDE_DIRS "."
    REQUIRES
        nvs_flash
        esp_wifi
        esp_netif
        esp_event
        spiffs
        esp_http_server
        driver
        esp_timer
        bt
        freertos
)
