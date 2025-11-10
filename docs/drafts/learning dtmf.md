---
comments: true
draft: true
date: 2025-01-17
tags: [fft, dsp]
---

# 非常渴望学习FFT
傅里叶变换将时域采样信号转换成许多频域（频谱列表）信号，基于“任何波形都能用N个正弦波来模拟”， N越大模拟的效果越好。

注意：
1. 采样是按一定的频率将连续的波形(模拟信号)转换成一组`值`（数字信号）， 这个值的类型可以是signed integer 

从简单开始，先学习`goertzel`算法， 

## goertzel
下面的代码是chatgpt生成的，用来检测一个指定频率的振幅。

```c
#include <stdio.h>
#include <math.h>

#define PI 3.14159265358979323846

// Function to calculate the Goertzel magnitude for a given frequency f_0
double goertzel(int16_t *data, int N, int f_0, int f_s) {
    // Calculate the Goertzel coefficient
    double k = 0.5 + ((N * f_0) / (double) f_s);  // N * f_0 / f_s
    double omega = 2 * PI * k;
    double coeff = 2 * cos(omega);
    
    // Variables for the Goertzel recurrence
    double s_prev = 0.0, s_prev2 = 0.0;
    double s_curr;
    
    // Loop through all samples and apply the Goertzel recurrence
    for (int i = 0; i < N; i++) {
        s_curr = data[i] + coeff * s_prev - s_prev2;
        s_prev2 = s_prev;
        s_prev = s_curr;
    }
    
    // Calculate the magnitude of the frequency f_0
    double magnitude = sqrt(s_prev2 * s_prev2 + s_prev * s_prev - coeff * s_prev * s_prev2);
    
    return magnitude;aa
}

int main() {
    // Example PCM data (16-bit signed integers)
    int16_t pcm_data[] = {100, 200, 150, -50, 200, -150, 100, -100};  // Replace with actual PCM data
    int N = sizeof(pcm_data) / sizeof(pcm_data[0]);  // Number of samples
    int f_s = 44100;  // Sample rate (Hz)
    int f_0 = 1000;    // Frequency to detect (Hz)
    
    // Calculate the magnitude of the specified frequency using Goertzel algorithm
    double magnitude = goertzel(pcm_data, N, f_0, f_s);
    
    // Print the magnitude
    printf("Magnitude at %d Hz: %f\n", f_0, magnitude);
    
    return 0;
}
```