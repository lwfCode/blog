---
title: Go+cloudConvert 合并不同格式文件形成新的pdf
date: 2025-05-09 17:27:31
tags: Go
categories: 后端
top: 27
---

### 前言

cloudConvert是一个在线文件转换工具，支持将多种文件格式转换为PDF格式。它的主要特点包括：
- 支持多种文件格式的转换，包括但不限于：Word、Excel、PPT、PDF、图片、音频、视频等。
- 支持批量转换，用户可以同时转换多个文件。
- 支持在线转换，用户可以直接在浏览器中进行转换，无需下载客户端。

ps:需要注册一个账号才能使用，免费的账号有一定的限制。

正好最近在做一个项目，需要将不同格式的文件合并成一个pdf文件，所以就用了cloudConvert的api。

#### 代码

file/main.go
```
package main

import (
	"cloudconvert-cli/task"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
)

type MergeRequest struct {
	FileURLs []string `json:"fileURLs"`
}

func mergeHandler(w http.ResponseWriter, r *http.Request) {
	var reqBody MergeRequest
	if err := json.NewDecoder(r.Body).Decode(&reqBody); err != nil {
		http.Error(w, "Invalid input format", http.StatusBadRequest)
		return
	}
	fileURLs := reqBody.FileURLs

	if len(fileURLs) == 0 {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusOK)
		json.NewEncoder(w).Encode(map[string]string{
			"error":   "fileURLs empty",
			"message": "Please provide at least one valid file URL.",
		})
		return
	}

	go func() {
		_, _, _, err := task.Run(fileURLs)
		if err != nil {
			http.Error(w, fmt.Sprintf("Job failed: %v", err), http.StatusInternalServerError)
			return
		}
	}()

	response := map[string]string{
		"code":    "200",
		"message": "Job successfully",
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}
func main() {
	http.HandleFunc("/merge", mergeHandler)
	fmt.Println("running on:8083")
	log.Fatal(http.ListenAndServe(":8083", nil))
}
```

task/task.go
```
package task

import (
	"bytes"
	"crypto/rand"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"time"
)

const (
	mdyApi          = "https://you.domain.com"
	apiKey          = "you api key"
	cloudConvertAPI = "https://api.cloudconvert.com/v2/jobs"
	saveDirectory = "/data/public/storage/pdf/"
)

// --- 数据结构 ---
type Task struct {
	Operation string   `json:"operation"`
	Input     []string `json:"input,omitempty"`
	URL       string   `json:"url,omitempty"`
	Format    string   `json:"output_format,omitempty"`
}

type JobRequest struct {
	Tasks map[string]Task `json:"tasks"`
}

type JobResponse struct {
	Data struct {
		ID     string `json:"id"`
		Status string `json:"status"`
		Tasks  []struct {
			ID     string `json:"id"`
			Name   string `json:"name"`
			Status string `json:"status"`
		} `json:"tasks"`
	} `json:"data"`
}

type ExportResponse struct {
	Data struct {
		Status string `json:"status"`
		Result struct {
			Files []struct {
				URL string `json:"url"`
			} `json:"files"`
		} `json:"result"`
	} `json:"data"`
}

// --- 创建合并任务 ---
func createJob(fileURLs []string) (string, string, error) {
	tasks := make(map[string]Task)

	// 构建导入任务
	for i, fileURL := range fileURLs {
		taskName := fmt.Sprintf("import-%d", i+1)
		tasks[taskName] = Task{
			Operation: "import/url",
			URL:       fileURL,
		}
	}

	// 构建合并任务
	inputs := make([]string, len(fileURLs))
	for i := 0; i < len(fileURLs); i++ {
		inputs[i] = fmt.Sprintf("import-%d", i+1)
	}
	tasks["merge"] = Task{
		Operation: "merge",
		Input:     inputs,
		Format:    "pdf",
	}

	jobRequest := JobRequest{Tasks: tasks}
	requestBody, err := json.Marshal(jobRequest)
	if err != nil {
		return "", "", err
	}

	req, err := http.NewRequest("POST", cloudConvertAPI, bytes.NewBuffer(requestBody))
	if err != nil {
		return "", "", err
	}
	req.Header.Set("Authorization", "Bearer "+apiKey)
	req.Header.Set("Content-Type", "application/json")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return "", "", err
	}
	defer resp.Body.Close()

	var jobResponse JobResponse
	if err := json.NewDecoder(resp.Body).Decode(&jobResponse); err != nil {
		return "", "", err
	}

	// 返回 JobId 和taskId
	return jobResponse.Data.ID, jobResponse.Data.Tasks[len(jobResponse.Data.Tasks)-1].ID, nil
}

// --- 创建导出任务 ---
func createExportTask(taskID string) (string, error) {
	exportTask := map[string]interface{}{
		"tasks": map[string]interface{}{
			"export-1": map[string]interface{}{
				"operation": "export/url",
				"input":     taskID,
			},
		},
	}

	requestBody, err := json.Marshal(exportTask)
	if err != nil {
		return "", fmt.Errorf("failed to marshal export task: %v", err)
	}

	// 调 CloudConvert api
	req, err := http.NewRequest("POST", cloudConvertAPI, bytes.NewBuffer(requestBody))
	if err != nil {
		return "", fmt.Errorf("failed to create export request: %v", err)
	}
	req.Header.Set("Authorization", "Bearer "+apiKey)
	req.Header.Set("Content-Type", "application/json")

	// 发送请求
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return "", fmt.Errorf("failed to send export request: %v", err)
	}
	defer resp.Body.Close()

	// 解析响应
	var exportResponse JobResponse
	if err := json.NewDecoder(resp.Body).Decode(&exportResponse); err != nil {
		return "", fmt.Errorf("failed to decode export response: %v", err)
	}

	// 获取任务 ID
	if len(exportResponse.Data.Tasks) == 0 {
		return "", fmt.Errorf("no export tasks found in response")
	}
	return exportResponse.Data.Tasks[0].ID, nil
}

// --- 轮询导出任务并下载 ---
func downloadFile(taskID string) {
	for {
		// 请求任务状态
		url := fmt.Sprintf("https://api.cloudconvert.com/v2/tasks/%s", taskID)
		req, err := http.NewRequest("GET", url, nil)
		if err != nil {
			log.Fatal("Error creating request:", err)
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			log.Fatal("Error fetching export task details:", err)
		}
		defer resp.Body.Close()

		// 解析响应
		var exportResponse ExportResponse
		if err := json.NewDecoder(resp.Body).Decode(&exportResponse); err != nil {
			log.Fatal("Error decoding export response:", err)
		}

		// 如果有url，则下载
		if len(exportResponse.Data.Result.Files) > 0 {
			fileURL := exportResponse.Data.Result.Files[0].URL
			fmt.Printf("Download URL found: %s\n", fileURL)

			// 下载文件
			response, err := http.Get(fileURL)
			if err != nil {
				log.Fatal("Error downloading file:", err)
			}
			defer response.Body.Close()

            //目录加日期以便区分
			date := time.Now().Format("20060102")
			saveDirectory := saveDirectory + date + "/"

			//文件名加随机数
			randomName, _ := GenerateRandomString(12)
			if _, err := os.Stat(saveDirectory); os.IsNotExist(err) {
				os.MkdirAll(saveDirectory, 0755)
			}
			// 拼接保存路径
			savePath := saveDirectory + randomName + ".pdf"

			// 保存到本地
			out, err := os.Create(savePath)
			if err != nil {
				log.Fatal("Error creating file:", err)
			}
			defer out.Close()

			_, err = io.Copy(out, response.Body)
			if err != nil {
				log.Fatal("Error saving file:", err)
			}

			fmt.Printf("文件成功保存到: %s\n", savePath)

			url := mdyApi + "/storage/pdf/" + date + "/" + randomName + ".pdf"
			WriteToMding(url)
			return
		}
		//轮询
		fmt.Println("url ready, retrying in 3 seconds...")
		time.Sleep(3 * time.Second)
	}
}

// 生成随机数
func GenerateRandomString(length int) (string, error) {
	// 每个字节是 2 个 16 进制字符，所以需要 length/2 的字节数
	bytes := make([]byte, length/2)
	if _, err := rand.Read(bytes); err != nil {
		return "", err
	}
	return hex.EncodeToString(bytes), nil
}

func Run(fileURLs []string) (string, string, string, error) {

	jobID, taskID, err := createJob(fileURLs)
	if err != nil {
		log.Fatalf("Failed to create job: %v", err)
	}

	exportTaskID, err := createExportTask(taskID)
	if err != nil {
		log.Fatalf("Failed to create export task: %v", err)
	}

	downloadFile(exportTaskID)
	return jobID, taskID, exportTaskID, err
}

//写入数据库/第三方平台/其他等，此处是根据你的业务需求进行处理  --可选
func WriteToMding(url string) {
	type Control struct {
		ControlId string `json:"controlId"`
		Value     string `json:"value"`
	}
	type RequestBody struct {
		AppKey      string    `json:"appKey"`
		Sign        string    `json:"sign"`
		WorksheetId string    `json:"worksheetId"`
		Controls    []Control `json:"controls"`
	}

	api := mdyApi + "/api/v2/open/worksheet/addRow"
	currentTime := time.Now()
	date := currentTime.Format("2006-01-02 15:04:05")
	// 构建请求体
	requestBody := RequestBody{
		AppKey:      "you appkey",
		Sign:        "you sign",
		WorksheetId: "rwjlb",
		Controls: []Control{
			{
				ControlId: "title",
				Value:     "测试标题",
			},
			{
				ControlId: "created_time",
				Value:     date,
			},
			{
				ControlId: "file",
				Value:     url,
			},
		},
	}
	jsonData, err := json.Marshal(requestBody)
	if err != nil {
		log.Fatalf("JSON 序列化失败: %v", err)
	}

	fmt.Println("请求参数:")
	fmt.Println(string(jsonData))

	req, err := http.NewRequest("POST", api, bytes.NewBuffer(jsonData))
	if err != nil {
		log.Fatalf("创建请求失败: %v", err)
	}
	req.Header.Set("Content-Type", "application/json")

	client := &http.Client{}
	resp, err := client.Do(req)
	if err != nil {
		log.Fatalf("请求失败: %v", err)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		log.Fatalf("读取响应失败: %v", err)
	}
	//打印结果
	fmt.Println(string(body))
}
```

### 最终实现结果

#### 命令行调用

![alt text](/images/file/result.jpg)

#### 合并后的pdf文件
![alt text](/images/file/result1.jpg)

最后附上cloudConvert官网地址：https://cloudconvert.com
