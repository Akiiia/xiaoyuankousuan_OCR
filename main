import time

import cv2
import numpy as np
from ppadb.client import Client as AdbClient
from cnocr import CnOcr
from openai import OpenAI


def is_color_close(pixel_bgr, target_bgr, threshold=50):
    """
    判断某个像素的BGR颜色是否接近目标颜色

    :param pixel_bgr: 像素点的 BGR 颜色值
    :param target_bgr: 目标 BGR 颜色值
    :param threshold: 阈值，颜色差异小于此值认为接近
    :return: boolean, 是否接近
    """
    b_diff = int(pixel_bgr[0]) - int(target_bgr[0])
    g_diff = int(pixel_bgr[1]) - int(target_bgr[1])
    r_diff = int(pixel_bgr[2]) - int(target_bgr[2])

    # 计算距离的平方，并比较平方值
    distance_squared = b_diff * b_diff + g_diff * g_diff + r_diff * r_diff
    threshold_squared = threshold * threshold

    return distance_squared < threshold_squared

def is_next_question():
    # 避免复制就直接在这里截图吧
    result = device.screencap()
    img_array = np.frombuffer(result, dtype=np.uint8)
    image = cv2.imdecode(img_array, cv2.IMREAD_COLOR)

    next_question_coords = (377, 996)

    target_bgr = [239, 161, 79]

    # 获取指定坐标的颜色值
    next_question_color = image[next_question_coords[1], next_question_coords[0]]
    return is_color_close(next_question_color, target_bgr)

def check_options_color(image):
    # 选项坐标
    first_option_coords = (64, 1080)
    second_option_coords = (65, 1200)

    # 目标颜色 BGR
    target_bgr = [255, 224, 179]

    # 如果第一个选项点为蓝色，则代表有四个选项
    first_option_color = image[first_option_coords[1], first_option_coords[0]]  # 注意[y,x]顺序
    if is_color_close(first_option_color, target_bgr):
        return 4

    second_option_color = image[second_option_coords[1], second_option_coords[0]]  # 注意[y,x]顺序
    if is_color_close(second_option_color, target_bgr):
        return 3

    return 2  # 如果两者都无，返回2


def get_question_text():  # 你应该在 答题开始和检测到下一题 后执行

    left, top, right, bottom = 45, 495, 855, 1555
    cropped_image = image[top:bottom, left:right]

    cropped_image_gray = cv2.cvtColor(cropped_image, cv2.COLOR_BGR2GRAY)

    # cv2.imshow('Input to OCR', cropped_image_gray)
    # cv2.waitKey(0)

    out = ocr.ocr(cropped_image_gray)

    text_string = " ".join(item['text'] for item in out)

    print("OCR 识别结果:")
    print(text_string)

    return text_string


def get_chat_response(str):
    str = "回答以下题目，仅返回ABCD四个字符之一，不需要额外回答，顺序排序。" + str

    client = OpenAI(
        # 若没有配置环境变量，请用百炼API Key将下行替换为：api_key="sk-xxx",
        api_key="",
        base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    )

    completion = client.chat.completions.create(
        model="qwen-plus",  # 模型列表：https://help.aliyun.com/zh/model-studio/getting-started/models
        messages=[
            {'role': 'user', 'content': str}
        ],
    )

    message_content = completion.choices[0].message.content
    return message_content


def click_option(option_count, clicked_option):
    print(f"选项数量 {option_count}，点击选项 {clicked_option}")
    if option_count == 4:
        coordinate_list = [(450, 1065), (450, 1215), (450, 1350), (450, 1500)]
    elif option_count == 3:
        coordinate_list = [(450, 1215), (450, 1350), (450, 1500)]
    else:
        coordinate_list = [(450, 1350), (450, 1500)]

    option_map = {'A': 0, 'B': 1, 'C': 2, 'D': 3}

    # 判断入参是否正确
    if clicked_option in option_map and option_map[clicked_option] < option_count:
        index = option_map[clicked_option]
        x, y = coordinate_list[index]
        # print(f'input tap {x} {y}')
        device.shell(f'input tap {x} {y}')
    else:
        print("无效的选项")

# 初始化 OCR
ocr = CnOcr()

# 初始化 ADB 客户端
client = AdbClient(host="127.0.0.1", port=5037)

devices = client.devices()
if not devices:
    print("没有连接设备")
    exit(1)
device = devices[0]

while True:
    input("请输入以继续: ")

    for i in range(8):
        print(f"正在识别... （第 {i + 1} 次）")

        result = device.screencap()

        img_array = np.frombuffer(result, dtype=np.uint8)
        image = cv2.imdecode(img_array, cv2.IMREAD_COLOR)

        # 检查选项颜色并返回对应值
        result = check_options_color(image)
        # print("选项数量：", result)

        click_option(result, get_chat_response(get_question_text()))
        # click_option(result, get_chat_response_openai(get_question_text()))
        while not is_next_question():
            # 死循环直到检测到下一题再退出
            if i + 1 == 8:
                print("已经是最后一题")
                break

        time.sleep(0.8)
        print("切换到下一题")
