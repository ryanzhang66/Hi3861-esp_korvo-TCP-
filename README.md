# Hi3861+esp_korvo （TCP）

## 前言
在进行开发两者的TCP通信时遇到了一些问题，在这里我将提供一些解决方案。因为两者都是RTOS任务调度机制，所以开发难度还是不小的。

## hi3861TCP服务端实现
### 实现步骤
我这里不过多介绍，具体步骤可以学习bearpi官方案例。
1. 先创建socket的一系列操作
2. 在监听到客户端后进行后续操作

### TCP任务函数
我一开始设置延迟20秒是为了我在源码当中的其他任务争取时间
```c
static void TCPServerTask(void)
{
    osDelay(2000U);
    // 在sock_fd 进行监听，在 new_fd 接收新的链接
    int sock_fd;

    // 服务端地址信息
    struct sockaddr_in server_sock;

    // 客户端地址信息
    struct sockaddr_in client_sock, *cli_addr;
    int sin_size;

    // 创建socket
    if ((sock_fd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("socket is error\r\n");
        return;
    }

    bzero(&server_sock, sizeof(server_sock));
    server_sock.sin_family = AF_INET;
    server_sock.sin_addr.s_addr = htonl(INADDR_ANY);
    server_sock.sin_port = htons(CONFIG_CLIENT_PORT);

    // 调用bind函数绑定socket和地址
    if (bind(sock_fd, (struct sockaddr *)&server_sock, sizeof(struct sockaddr)) == -1) {
        return;
    }

    // 调用listen函数监听(指定port监听)
    if (listen(sock_fd, TCP_BACKLOG) == -1) {
        return;
    }

    printf("start accept\n");

    // 调用accept函数从队列中
    while (1) {
        osMutexAcquire(muxe_id,osWaitForever);
        sin_size = sizeof(struct sockaddr_in);

        if ((new_fd = accept(sock_fd, (struct sockaddr *)&client_sock, (socklen_t *)&sin_size)) == -1) {
            perror("accept");
            continue;
        }

        cli_addr = malloc(sizeof(struct sockaddr));

        printf("accept addr\r\n");

        if (cli_addr != NULL) {
            int ret;
            if  (ret = memcpy_s(cli_addr, sizeof(struct sockaddr), &client_sock, sizeof(struct sockaddr)) != 0) {
                perror("memcpy is error\r\n");
                return;
            }
        }
        // 处理目标
        ssize_t ret;

        while (1) {
            memset_s(recvbuf, sizeof(recvbuf), 0, sizeof(recvbuf));
            if ((ret = recv(new_fd, recvbuf, sizeof(recvbuf), 0)) == -1) {
                printf("recv error \r\n");
            }
            printf("recv :%s\r\n", recvbuf);
            sleep(TASK_DELAY_2S);
        }
        close(new_fd);
    }
}
```

## esp_korvo TCP客户端实现

### 实现步骤

1. wifi连接函数执行
2. 将TCP初始化的设置放在main函数中。

### 实现代码
具体TCP实现的代码我就不一一解读了，因为在我的esp_korvo_chinese_tts仓库中已经详细说明。
```c
int app_main()
{
    // 创建Wi-Fi任务
    xTaskCreate(wifiTask, "Wifi Task", 4096, NULL, 1, NULL);

    #if defined CONFIG_ESP32_S3_EYE_BOARD
    printf("Not Support esp32-s3-eye board\n");
    return 0;
#endif

    ESP_ERROR_CHECK(esp_board_init(16000, 1, 16));
#ifdef SDCARD_OUTPUT_ENABLE
    ESP_ERROR_CHECK(esp_sdcard_init("/sdcard", 10));
    FILE* fp=fopen("/sdcard/URAT.pcm", "w+");
    if(fp==NULL)
        printf("can not open file!\n");

    //sample rate:16000Hz, int16, mono
    void * wav_encoder=wav_encoder_open("/sdcard/prompt.wav", 16000, 16, 1);
    void * urat_wav_encoder=wav_encoder_open("/sdcard/URAT.wav", 16000, 16, 1);

#endif


    char host_ip[] = HOST_IP_ADDR;
    int addr_family = 0;
    int ip_protocol = 0;

    while (1) {
        struct sockaddr_in dest_addr;
        dest_addr.sin_addr.s_addr = inet_addr(host_ip);
        dest_addr.sin_family = AF_INET;
        dest_addr.sin_port = htons(PORT);

        xEventGroupWaitBits(wifiEvent, wifiConnectedBit, pdFALSE, pdFALSE, portMAX_DELAY);

        int sock =  socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
        if (sock < 0) {
            ESP_LOGE(TAG, "Unable to create socket: errno %d", errno);
            break;
        }
        ESP_LOGI(TAG, "Socket created, connecting to %s:%d", host_ip, PORT);

        int err = connect(sock, (struct sockaddr *)&dest_addr, sizeof(dest_addr));
        if (err != 0) {
            ESP_LOGE(TAG, "Socket unable to connect: errno %d", errno);
            break;
        }
        ESP_LOGI(TAG, "Successfully connected");


        /*** 1. create esp tts handle ***/
        // initial voice set from separate voice data partition

        const esp_partition_t* part=esp_partition_find_first(ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_ANY, "voice_data");
        if (part==NULL) { 
            printf("Couldn't find voice data partition!\n"); 
            return 0;
        } else {
            printf("voice_data paration size:%d\n", part->size);
            checkFreeHeapSize();
        }
        void* voicedata;

        esp_partition_mmap_handle_t mmap;
        err=esp_partition_mmap(part, 0, part->size, ESP_PARTITION_MMAP_DATA, &voicedata, &mmap);

        if (err != ESP_OK) {
            printf("Couldn't map voice data partition!\n"); 
            return 0;
        }
        esp_tts_voice_t *voice=esp_tts_voice_set_init(&esp_tts_voice_template, (int16_t*)voicedata); 
        
        esp_tts_handle_t *tts_handle=esp_tts_create(voice);
        int i = 0;       

        while (1) {
            int err = send(sock, payload, strlen(payload), 0);
            if (err < 0) {
                ESP_LOGE(TAG, "Error occurred during sending: errno %d", errno);
                break;
            }

            int len = recv(sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
            // Error occurred during receiving
            if (len < 0) {
                ESP_LOGE(TAG, "recv failed: errno %d", errno);
                break;
            }
            // Data received
            else {
                rx_buffer[len] = 0; // Null-terminate whatever we received and treat like a string
                ESP_LOGI(TAG, "Received %d bytes from %s:", len, host_ip);
                ESP_LOGI(TAG, "%s", rx_buffer);

                //memset(rx_buffer,0,sizeof(rx_buffer));
                if (i == 0)
                {
                    /*** 2. play prompt text ***/
                    char *prompt1="欢迎使用手语之声手套";  
                    printf("%s\n", prompt1);
                    if (esp_tts_parse_chinese(tts_handle, prompt1)) {
                            int len[1]={0};
                            do {
                                short *pcm_data=esp_tts_stream_play(tts_handle, len, 3);
                                esp_audio_play(pcm_data, len[0]*2, portMAX_DELAY);
                            } while(len[0]>0);
                    }
                    esp_tts_stream_reset(tts_handle);
                    i = 1;
                }
            }
        }

        if (sock != -1) {
            ESP_LOGE(TAG, "Shutting down socket and restarting...");
            shutdown(sock, 0);
            close(sock);
        }
    }
    return 0;
}
```

## 任务逻辑实现

一开始我将TCP通信任务单独作为一个task，但是这样我在测试的时候发现了一些问题，两者之间没有办法进行通信。原因是任务之间的调度出现了问题

这里我将调度好的方案简述一下：

### Hi3861端调度方案
- 首先TCP任务保持一个线程不变，但是在接收到客户端连接返回的参数需要设置一个全局变量，因为我需要在其他任务函数中满足条件后发送通信数据。
    ```c
    int new_fd;  //全局变量
    ``` 
- 在逻辑任务函数中，满足条件发送数据。
    ```c
    if (flex_1 < 630 && flex_2 < 990 && flex_3 < 800 && flex_4 < 700 && flex_5 < 670)
                {
                    // 使用全局变量（客户端参数）new_fd
                    if ((ret = send(new_fd, buf_ni, strlen(buf_ni) + 1, 0)) == -1) {
                        perror("send : ");
                    }
                } 

    else if ...
    ``` 
- 两个任务的优先级是一样的，均为24
    ```c
    osThreadAttr_t attr;

    attr.name = "TCPServerTask";
    attr.attr_bits = 0U;
    attr.cb_mem = NULL;
    attr.cb_size = 0U;
    attr.stack_mem = NULL;

    attr.stack_size = TASK_STACK_SIZE;
    attr.priority = 24;
    attr.name = "TCPServerTask";
    if (osThreadNew((osThreadFunc_t)TCPServerTask, NULL, &attr) == NULL) {
        printf("[TCPServerDemo] Failed to create TCPServerTask!\n");
    }

    attr.stack_size = CLOUD_TASK_STACK_SIZE;
    attr.priority = 25;
    attr.name = "CloudMainTaskEntry";
    if (osThreadNew((osThreadFunc_t)CloudMainTaskEntry, NULL, &attr) == NULL) {
        printf("Failed to create CloudMainTaskEntry!\n");
    }

    attr.stack_size = SENSOR_TASK_STACK_SIZE;
    attr.priority = 24;
    attr.name = "SensorTaskEntry";
    if (osThreadNew((osThreadFunc_t)flex_SensorTask, NULL, &attr) == NULL) {
        printf("Failed to create SensorTaskEntry!\n");
    }
    ``` 

### ESP端调度方案
- 将TCP任务和逻辑任务放在一起。  
- 先完成初始化。  
    ```c
    char host_ip[] = HOST_IP_ADDR;
    int addr_family = 0;
    int ip_protocol = 0;

    while (1) {
        struct sockaddr_in dest_addr;
        dest_addr.sin_addr.s_addr = inet_addr(host_ip);
        dest_addr.sin_family = AF_INET;
        dest_addr.sin_port = htons(PORT);

        xEventGroupWaitBits(wifiEvent, wifiConnectedBit, pdFALSE, pdFALSE, portMAX_DELAY);

        int sock =  socket(AF_INET, SOCK_STREAM, IPPROTO_IP);
        if (sock < 0) {
            ESP_LOGE(TAG, "Unable to create socket: errno %d", errno);
            break;
        }
        ESP_LOGI(TAG, "Socket created, connecting to %s:%d", host_ip, PORT);

        int err = connect(sock, (struct sockaddr *)&dest_addr, sizeof(dest_addr));
        if (err != 0) {
            ESP_LOGE(TAG, "Socket unable to connect: errno %d", errno);
            break;
        }
        ESP_LOGI(TAG, "Successfully connected");


        /*** 1. create esp tts handle ***/
        // initial voice set from separate voice data partition

        const esp_partition_t* part=esp_partition_find_first(ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_ANY, "voice_data");
        if (part==NULL) { 
            printf("Couldn't find voice data partition!\n"); 
            return 0;
        } else {
            printf("voice_data paration size:%d\n", part->size);
            checkFreeHeapSize();
        }
        void* voicedata;

        esp_partition_mmap_handle_t mmap;
        err=esp_partition_mmap(part, 0, part->size, ESP_PARTITION_MMAP_DATA, &voicedata, &mmap);

        if (err != ESP_OK) {
            printf("Couldn't map voice data partition!\n"); 
            return 0;
        }
        esp_tts_voice_t *voice=esp_tts_voice_set_init(&esp_tts_voice_template, (int16_t*)voicedata); 
        
        esp_tts_handle_t *tts_handle=esp_tts_create(voice);
    ```  
- 在while循环中持续检测接收的消息是否符合条件。
    ```c
    while (1) {
            int err = send(sock, payload, strlen(payload), 0);
            if (err < 0) {
                ESP_LOGE(TAG, "Error occurred during sending: errno %d", errno);
                break;
            }

            int len = recv(sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
            // Error occurred during receiving
            if (len < 0) {
                ESP_LOGE(TAG, "recv failed: errno %d", errno);
                break;
            }
            // Data received
            else {
                rx_buffer[len] = 0; // Null-terminate whatever we received and treat like a string
                ESP_LOGI(TAG, "Received %d bytes from %s:", len, host_ip);
                ESP_LOGI(TAG, "%s", rx_buffer);

                //memset(rx_buffer,0,sizeof(rx_buffer));
                if (i == 0)
                {
                    /*** 2. play prompt text ***/
                    char *prompt1="欢迎使用...";  
                    printf("%s\n", prompt1);
                    if (esp_tts_parse_chinese(tts_handle, prompt1)) {
                            int len[1]={0};
                            do {
                                short *pcm_data=esp_tts_stream_play(tts_handle, len, 3);
                                esp_audio_play(pcm_data, len[0]*2, portMAX_DELAY);
                            } while(len[0]>0);
                    }
                    esp_tts_stream_reset(tts_handle);
                    i = 1;
                }

                else if (strcmp(rx_buffer,"ni") == 0)
                {
                    char *prompt2="你";  
                    printf("%s\n", prompt2);
                    if (esp_tts_parse_chinese(tts_handle, prompt2)) {
                            int len[1]={0};
                            do {
                                short *pcm_data=esp_tts_stream_play(tts_handle, len, 3);
                                esp_audio_play(pcm_data, len[0]*2, portMAX_DELAY);
                            } while(len[0]>0);
                    }
                    esp_tts_stream_reset(tts_handle);
                    memset(rx_buffer,0,sizeof(rx_buffer));  //接收完清除接收数据
                }
    ```   

## 总结

任务调度不好弄，还要再继续努力！