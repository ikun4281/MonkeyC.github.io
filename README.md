# 基于coze搭建的智能体“一图批改“
## 运行示例
示例1原图：
![示例1原图](https://github.com/ikun4281/MonkeyC.github.io/blob/main/%E7%A4%BA%E4%BE%8B_%E5%8E%9F%E5%9B%BE/image1.jpg)

示例1标注：
![示例1标注](https://github.com/ikun4281/MonkeyC.github.io/blob/main/%E7%A4%BA%E4%BE%8B_%E6%A0%87%E6%B3%A8/image1.png)

更多示例可查看文件夹，使用体验可访问：[点击跳转到Coze链接](https://www.coze.cn/s/_uPTlC1UjJs/)

## 一、智能体配置

### 1. 模型

豆包·2.0·pro

### 2. 开场白

```
你好！我是作文批改智能体 ✨
上传一张作文图片，我会为你自动批改并返回批改结果链接。
支持填写姓名，方便识别作文。你可以上传一张图片，然后在对话框中输入姓名并发送。

现在上传你的作文吧！
```

### 3. 人设与回复逻辑

```

## 角色

你是作文批改智能体，**只做两件事**：

1. 调用批改工作流工具
2. 按固定格式返回链接

## 输入

* 用户上传图片：`user\_image`
* 用户输入姓名：`name`（可不填）

## 执行规则（必须严格照做，不许多想）

1. **只要用户上传图片，直接调用工具：`ts-Pigai-Pigai`**
参数传：

   * `user\_image`：用户上传的图片
   * `name`：用户输入的名字
2. **不许存记忆、不许设关键词、不许转图片、不许下载图片**
3. 工具返回 `output`（URL）后，**只按下面模板回复**，不许加任何其他内容。

## 回复模板（固定死，不许改）

* 如果有名字：
**{name}的作文已完成批改，请点击链接查看:{output}**
* 如果没有名字：
**你的作文已完成批改，请点击链接查看:{output}**

## 禁止行为（一条都不许破）

* 不许思考“要不要存记忆”
* 不许思考“要不要转图片”
* 不许思考“要不要调用其他工具”
* 不许加多余话
* 不许解释内容
* 不许发图片，只发链接文本
```

## 二、工作流配置

### 1. 节点1：开始

输入：

|变量名|类型|
|-|:-:|
|name|String|
|user_image|String|


### 2. 节点2：high_precision_recognition
 商店引用的OCR插件，基于阿里云百炼最新OCR视觉理解模型qwen-vl-ocr-latest，内置多种提取工具，能够逐行返回句子坐标。
输入：
|变量名|类型|值|
|-|-|:-:|
|api_key|String|百炼api|
|image|String|节点1：user_image|

输出（取用部分）：
|变量名|类型|值|
|-|-|:-:|
|text|String|OCR识别文本及坐标|

### 3. 节点3：大模型
#### 模型
DeepSeek-V3.2 关闭深度思考 最大回复长度=10000

#### 输入：
|变量名|类型|值|
|-|-|:-:|
|text|String|节点2：text|

#### 系统提示词
```
## R - Role（角色定义）
你是专业的小学生作文批改AI，具备**零误差字符计数能力**和OCR坐标精准计算能力。必须严格按规则逐字符计数（禁止多数/少数），严格按公式计算坐标，适配方格纸标点排版规则。

---

## C - Context（输入格式与核心规则）
`{{text}}` 为作文OCR识别结果，JSON数组，每个文本块结构：
- `rotate_rect`: [cx, cy, w, h, angle]
  - cx：文本块中心横坐标；cy：中心纵坐标；w：总宽度；h：总高度；angle：旋转角度
- `text`: 文本块完整内容（**每个可见字符（含标点）独立计数，无隐藏字符**）

### 角度场景定义（仅两类）
1. 横排/微倾斜：angle ∈ {0,1,2,3,4,5}
2. 竖排：angle∈ {87,88,89,90,91,92}

---

# 【核心铁律 · 零误差字符计数规则（强制100%执行）】
## 1. 绝对禁止行为
- 禁止多数1个字符、禁止少数字符
- 禁止数到错误词就停止计数
- 禁止凭感觉估算、禁止主观加/减字符数

## 2. 强制字符拆解+双向校验流程（必须逐步骤执行）
### 步骤1：字符拆解（可视化拆分）
将文本块`text`内容**逐个字符拆分成列表**，列表中每一个元素对应1个字符（无合并、无遗漏、无空元素）。
✅ 正确示例：
文本："果被男孩推了过去，我所性就吃"
拆解列表：["果","被","男","孩","推","了","过","去","，","我","所","性","就","吃"]
❌ 错误示例：
- 多1个：["果","被","男","孩","推","了","过","去","，","我","所","性","就","吃",""]（末尾空字符）
- 少1个：["果","被","男","孩","推","了","过","去","，","我","所","性","就"]
- 合并字符：["果被","男","孩"...]

## 3. 禁止输出幻觉
只改真实存在的错别字 / 用词错误，禁止照抄示例，必须保证错字错词存在于文本块。

### 步骤2：计数（唯一标准）
原始总字符数 = 拆解列表的**长度**（len(列表)），仅以此为依据，禁止其他计数方式。

### 步骤3：双向校验（必须通过）
校验1：拆解列表长度 = 肉眼逐字符数的结果（无争议）；
校验2：列表中无空字符、无合并字符、无重复字符；
⚠️ 任一校验不通过 → 重新拆解计数，直到完全匹配。

## 3. 方格纸字数修正（仅横排/微倾斜）
触发条件（**必须同时满足**）：
1. angle ∈ {0,1,2,3,4,5}；
2. 原始总字符数 = 5×k + 1（k为非负整数，如1=5×0+1、6=5×1+1）；
3. 拆解列表最后1个元素是标点（标点列表：。，！？、：；""''（）【】《》）。

修正规则：
- 满足 → 修正后n = 原始总字符数 − 1；
- 不满足 → 修正后n = 原始总字符数。

✅ 修正示例：
文本："比手掌还大的哦子，还有世界上最美丽的蝴蝶。"
拆解列表长度=21，最后1个字符是"。"，21=5×4+1 → 修正后n=20。

竖排angle∈ {87,88,89,90,91,92}**不做任何修正**，n直接等于拆解列表长度。

---

# 【坐标计算公式 · 零误差执行】
## 场景1：横排/微倾斜（angle ∈ {0,1,2,3,4,5}）
已知：
- rotate_rect = [cx₀, cy₀, w₀, h₀, angle]；
- 修正后总字数n；
- 错误词位置：拆解列表中第i～j个元素（i/j从1开始，对应列表索引+1）。

计算步骤（一步不可改）：
1. 单字长度 = round(w₀ / n, 1)（保留1位小数）；
2. new_cx = cx₀ − w₀/2 + 单字长度 × (i + j − 1) / 2；
3. new_cy = cy₀（纵坐标不变）；
4. new_w = 单字长度 × (j − i + 1)；
5. new_h = h₀（高度不变）；
6. new_angle = angle（角度不变）。

最终坐标（四舍五入取整）：
[round(new_cx), new_cy, round(new_w), new_h, angle]

## 场景2：竖排angle∈ {87,88,89,90,91,92}
已知：
- rotate_rect = [cx₀, cy₀, w₀, h₀, 90]；
- 总字数n = 拆解列表长度（无修正）；
- 错误词位置：拆解列表中第i～j个元素（i/j从1开始）。

计算步骤（一步不可改）：
1. 单字长度 = round(h₀ / n, 1)（保留1位小数）；
2. new_cx = cx₀ − h₀/2 + 单字长度 × (i + j − 1) / 2；
3. new_cy = cy₀（纵坐标不变）；
4. new_w = w₀（宽度不变）；
5. new_h = 单字长度 × (j − i + 1)；
6. new_angle = 90（角度不变）。

最终坐标（四舍五入取整）：
[round(new_cx), new_cy, new_w, round(new_h), 90]

---

# 【坐标强制验证 · 不通过重算】
- 横排：new_cx 必须在 [cx₀ − w₀/2, cx₀ + w₀/2] 区间；
- 竖排：new_cx 必须在 [cx₀ − h₀/2, cx₀ + h₀/2] 区间；
⚠️ 验证不通过 → 优先检查「拆解列表长度（原始n）」是否正确，再检查i/j取值。

---

# 【批改输出规则】
1. grade（评分）：仅可选「良、良+、优、优+」，客观匹配作文质量；
2. feedback（评语）：仅可选「继续加油！、写得不错！、你很棒！、写得真好！」，与评分一一对应；
3. corrections（纠错）：
   - error：仅保留错误字符/最小错误词语（如“所性”“哦子”），禁止整句；
   - correct：仅写正确的字/词（如“索性”“蛾子”），无多余内容；
   - coord：严格按公式计算，所有数值为整数；
4. good_sentences（好句）：仅选取无语病、无错别字、表达生动的句子，附带原文本块rotate_rect；
5. bad_sentences（问题句）：仅保留「用词不当、语法错误、逻辑不通」的句子，**仅含错别字的句子禁止纳入**。

---

# 【标准输出格式（禁止修改）】
json{
  "good_sentences": [
    {
      "text": "",
      "coord": []
    }
  ],
  "bad_sentences": [
    {
      "text": "",
      "coord": [],
      "issue": ""
    }
  ],
  "corrections": [
    {
      "error": "",
      "correct": "",
      "coord": []
    }
  ],
  "grade": "",
  "feedback": ""
}
```

#### 输出：
|变量名|类型|值|
|-|-|:-:|
|res_text|String|大模型给出的评分评语、错词、好句及它们的坐标|

### 4. 节点4：代码
输入：
|变量名|类型|值|
|-|-|:-:|
|llm_output|String|节点3：res_text|
|user_image|String|节点1：user_image|

输出：
|变量名|类型|值|
|-|-|:-:|
|image_url|String|批注后图片的url|
|debug|String|报错信息|

由于coze环境无法导入PIL等第三方库，因此需要在云服务器构建api，然后通过代码节点调用来进行图像处理，代码节点Python代码：
```
import requests
import json
import re

async def main(args: Args) -> Output:
    params = args.params
    ret: Output = {
        "image_url": "",
        "debug": {
            "received_params": params,
            "llm_output_type": type(params.get("llm_output")).__name__,
            "llm_output_raw": params.get("llm_output"),
            "llm_output_cleaned": None,  # 清洗后的纯JSON字符串
            "llm_output_parsed": None,
            "user_image": params.get("user_image"),
            "error_msg": ""
        }
    }
    
    try:
        # ========== 1. 提取并清洗llm_output ==========
        llm_output = params.get("llm_output", "")
        ret["debug"]["llm_output_type"] = type(llm_output).__name__
        ret["debug"]["llm_output_raw"] = llm_output
        
        # 仅处理字符串类型的llm_output
        if isinstance(llm_output, str) and llm_output.strip():
            # 关键：提取```json和```之间的纯JSON内容
            json_match = re.search(r"```json\n(.*?)\n```", llm_output, re.DOTALL)
            if json_match:
                # 清洗掉代码块标记，得到纯JSON字符串
                cleaned_json = json_match.group(1).strip()
                ret["debug"]["llm_output_cleaned"] = cleaned_json
                # 解析清洗后的JSON
                llm_output = json.loads(cleaned_json)
                ret["debug"]["llm_output_parsed"] = llm_output
            else:
                ret["debug"]["error_msg"] = "未找到```json包裹的有效JSON内容"
                return ret
        else:
            ret["debug"]["error_msg"] = "llm_output不是非空字符串"
            return ret
        
        # ========== 2. 校验user_image ==========
        user_image = params.get("user_image", "").strip()
        ret["debug"]["user_image"] = user_image
        if not user_image:
            ret["debug"]["error_msg"] = "user_image为空"
            return ret
        
        # ========== 3. 调用批注API ==========
        API_URL = "输入你的API"
        response = requests.post(
            API_URL,
            json={"user_image": user_image, "llm_output": llm_output},
            headers={"Content-Type": "application/json"},
            timeout=20
        )
        
        # ========== 4. 解析结果 ==========
        api_result = response.json()
        ret["image_url"] = api_result.get("image_url", "")
        ret["debug"]["api_response"] = api_result
        
    except json.JSONDecodeError as e:
        ret["debug"]["error_msg"] = f"JSON解析失败：{str(e)}"
    except Exception as e:
        ret["debug"]["error_msg"] = f"整体错误：{str(e)}"
    
    return ret
```

云服务器端api代码：
```
from fastapi import FastAPI, HTTPException
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel
import json
import cv2
import numpy as np
from PIL import Image, ImageDraw, ImageFont
import os
import random
import urllib.request
from io import BytesIO

# ========== API基础配置 ==========
app = FastAPI()
# 1. 静态目录（存放批注后的图片，支持公网访问）
IMAGE_DIR = "/home/admin/annotation_api/static/images"
os.makedirs(IMAGE_DIR, exist_ok=True)
app.mount("/static", StaticFiles(directory="static"), name="static")
# 2. 服务器公网IP（替换为你的实际IP）
SERVER_IP = "你的实际IP"
# 3. Linux适配的字体路径（优先思源衬线体，最接近手写）
LINUX_HANDWRITING_FONT = "/usr/share/fonts/opentype/noto/NotoSerifCJK-Regular.ttc"
LINUX_NORMAL_FONT = "/usr/share/fonts/opentype/noto/NotoSansCJK-Regular.ttc"


# ========== 数据模型（API请求参数） ==========
class AnnotationRequest(BaseModel):
    user_image: str  # 图片URL（核心输入）
    llm_output: dict  # 大模型批注数据（含grade/feedback/ocr信息）


# ========== 核心工具函数 ==========
def extract_text_height(coord, img_w, img_h):
    """
    从大模型输出的coord中提取字高（th），并转换为像素
    coord格式：[cx, cy, w, h, angle]
    - angle 85-90°：字高=w（第三个值）
    - angle 0-5°：字高=h（第四个值）
    """
    if len(coord) != 5:
        return 47  # 默认字高（像素）

    cx_rel, cy_rel, w_rel, h_rel, angle = coord
    # 提取相对坐标的字高
    if 85 <= angle <= 90:
        th_rel = w_rel
    elif 0 <= angle <= 5:
        th_rel = h_rel
    else:
        th_rel = 55  # 兜底默认值

    # 相对坐标（0-999）转像素坐标
    th_pix = int(th_rel / 999 * max(img_w, img_h))  # 基于图片最大边转换，保证比例
    return max(th_pix, 10)  # 最小字高限制，避免过小


def get_base_text_height(llm_output, img_w, img_h):
    """从llm_output中获取基准字高（优先从corrections取，无则用默认）"""
    # 优先从corrections提取
    corrections = llm_output.get("corrections", [])
    if corrections:
        first_corr = corrections[0]
        coord = first_corr.get("coord", first_corr.get("rotate_rect", []))
        if coord:
            return extract_text_height(coord, img_w, img_h)

    # 其次从good_sentences提取
    good_sentences = llm_output.get("good_sentences", [])
    if good_sentences:
        first_good = good_sentences[0]
        coord = first_good.get("coord", [])
        if coord:
            return extract_text_height(coord, img_w, img_h)

    # 最后从bad_sentences提取
    bad_sentences = llm_output.get("bad_sentences", [])
    if bad_sentences:
        first_bad = bad_sentences[0]
        coord = first_bad.get("coord", [])
        if coord:
            return extract_text_height(coord, img_w, img_h)

    # 兜底默认值
    return 47


def draw_graffiti_wavy_line(img, start_point, end_point, color, thickness=5, amplitude=6, frequency=20):
    """绘制涂鸦风格波浪线（天蓝色、柔和、稍粗、低密度）"""
    x1, y1 = start_point
    x2, y2 = end_point
    dx = x2 - x1
    dy = y2 - y1
    distance = np.hypot(dx, dy)

    points = []
    for i in np.linspace(0, distance, int(distance / 3)):
        t = i / distance
        x = x1 + t * dx
        y = y1 + t * dy
        random_offset = random.uniform(-amplitude / 2, amplitude / 2)
        y += amplitude * np.sin(2 * np.pi * i / frequency) + random_offset
        points.append((int(round(x)), int(round(y))))

    for i in range(len(points) - 1):
        cv2.line(img, points[i], points[i + 1], color, thickness, lineType=cv2.LINE_AA)


def get_font_path(is_handwriting=True):
    """获取字体路径（适配Linux服务器，优先手写风格）"""
    if is_handwriting and os.path.exists(LINUX_HANDWRITING_FONT):
        return LINUX_HANDWRITING_FONT
    elif os.path.exists(LINUX_NORMAL_FONT):
        return LINUX_NORMAL_FONT
    return None


def draw_chinese_text(img, text, pos, font_size_pix, is_handwriting=True, color=(255, 0, 0)):
    """绘制中文文字（支持手写字体+精准字号）"""
    img_pil = Image.fromarray(cv2.cvtColor(img, cv2.COLOR_BGR2RGB))
    draw = ImageDraw.Draw(img_pil)

    font_path = get_font_path(is_handwriting)
    if font_path:
        font = ImageFont.truetype(font_path, font_size_pix, encoding="utf-8")
    else:
        font = ImageFont.load_default(size=font_size_pix)

    draw.text(pos, text, font=font, fill=color)
    return cv2.cvtColor(np.array(img_pil), cv2.COLOR_RGB2BGR)


def rel2pix(rel_value, img_size):
    """相对坐标（0-999）转像素坐标"""
    return int(rel_value / 999 * img_size)


def download_image_from_url(url):
    """从URL下载图片，返回OpenCV格式"""
    try:
        with urllib.request.urlopen(url, timeout=10) as resp:
            image_data = resp.read()
            nparr = np.frombuffer(image_data, np.uint8)
            img = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
            if img is None:
                raise ValueError("图片解码失败")
            return img
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"下载图片失败：{str(e)}")


def annotate_image(img, llm_output):
    """主批注函数（按字高自适应调整所有批注尺寸）"""
    img_h, img_w = img.shape[:2]
    edge_gap = 5  # 边缘间隙（仅用于与边界的距离，非坐标偏移）

    # ========== 核心：获取基准字高th（像素） ==========
    th = get_base_text_height(llm_output, img_w, img_h)

    # ========== 计算各批注的自适应尺寸 ==========
    grade_font_size = 4 * th  # grade尺寸：4*th
    feedback_font_size = 2 * th  # feedback尺寸：2*th
    correction_font_size = th  # corrections尺寸：th
    wave_amplitude = int(0.1 * th)  # good/bad_sentences振幅：0.1*th
    wave_thickness = max(1, int(0.12 * th))  # 波浪线粗细自适应

    # ------------------- 1. 绘制grade（紧贴右上角） -------------------
    grade = llm_output.get("grade", "")
    if grade:
        text_width = len(grade) * grade_font_size
        text_height = grade_font_size

        center_x = img_w - (text_width / 2) - edge_gap
        center_y = (text_height / 2) + edge_gap

        x = int(center_x - (text_width / 2))
        y = int(center_y - (text_height / 2))

        x = max(x, 0)
        x = min(x, img_w - text_width)
        y = max(y, 0)
        y = min(y, img_h - text_height)

        img = draw_chinese_text(
            img, grade, (x, y), grade_font_size,
            is_handwriting=True, color=(255, 0, 0)
        )

    # ------------------- 2. 绘制feedback（底部居中，2*th） -------------------
    feedback = llm_output.get("feedback", "")
    if feedback:
        text_width = len(feedback) * feedback_font_size
        text_height = feedback_font_size

        center_x = img_w / 2
        center_y = img_h - text_height

        x = int(center_x - (text_width / 2))
        y = int(center_y - (text_height / 2))

        x = max(x, 0)
        x = min(x, img_w - text_width)
        y = max(y, 0)
        y = min(y, img_h - text_height)

        img = draw_chinese_text(
            img, feedback, (x, y), feedback_font_size,
            is_handwriting=True, color=(255, 0, 0)
        )

    # ------------------- 3. 绘制corrections（红框 + 文字在正上方） -------------------
    corrections = llm_output.get("corrections", [])
    for corr in corrections:
        coord = corr.get("coord", corr.get("rotate_rect", []))
        if len(coord) != 5:
            continue
        cx_rel, cy_rel, w_rel, h_rel, angle = coord

        cx = rel2pix(cx_rel, img_w)
        cy = rel2pix(cy_rel, img_h)
        w = rel2pix(w_rel, img_w)
        h = rel2pix(h_rel, img_h)

        # 绘制红色旋转框
        rot_rect = ((cx, cy), (w, h), angle)
        box = cv2.boxPoints(rot_rect).astype(np.int32)
        cv2.drawContours(img, [box], 0, (0, 0, 255), 2, lineType=cv2.LINE_AA)

        # 绘制修正文字：放在红框 正上方 居中
        correct_text = corr.get("correct", "")
        if correct_text:
            # 计算文字宽度，实现居中
            text_w = len(correct_text) * correction_font_size
            # 位置：框的正上方，居中
            text_x = int(cx - text_w / 2)
            text_y = int(cy - h / 2 - correction_font_size - 4)  # 正上方

            # 边界保护
            text_x = max(edge_gap, min(text_x, img_w - edge_gap))
            text_y = max(edge_gap, min(text_y, img_h - edge_gap))

            img = draw_chinese_text(
                img, correct_text, (text_x, text_y), correction_font_size,
                is_handwriting=False, color=(255, 0, 0)
            )

    # ------------------- 4. 绘制good_sentences（天蓝色波浪线） -------------------
    sky_blue = (235, 206, 135)
    for sent in llm_output.get("good_sentences", []):
        coord = sent.get("coord", [])
        if len(coord) != 5:
            continue
        cx_rel, cy_rel, w_rel, h_rel, angle = coord

        cx = rel2pix(cx_rel, img_w)
        cy = rel2pix(cy_rel, img_h)
        w = rel2pix(w_rel, img_w)
        h = rel2pix(h_rel, img_h)

        rot_rect = ((cx, cy), (w, h), angle)
        box = cv2.boxPoints(rot_rect).astype(np.int32)
        line_y = box[:, 1].max() + 8
        line_x_min = box[:, 0].min()
        line_x_max = box[:, 0].max()

        line_y = max(min(line_y, img_h - edge_gap), edge_gap)
        line_x_min = max(line_x_min, edge_gap)
        line_x_max = min(line_x_max, img_w - edge_gap)

        draw_graffiti_wavy_line(
            img, (line_x_min, line_y), (line_x_max, line_y),
            color=sky_blue, thickness=wave_thickness, amplitude=wave_amplitude, frequency=22
        )

    # ------------------- 5. 绘制bad_sentences（黄色波浪线） -------------------
    yellow = (204, 255, 255)
    for sent in llm_output.get("bad_sentences", []):
        coord = sent.get("coord", [])
        if len(coord) != 5:
            continue
        cx_rel, cy_rel, w_rel, h_rel, angle = coord

        cx = rel2pix(cx_rel, img_w)
        cy = rel2pix(cy_rel, img_h)
        w = rel2pix(w_rel, img_w)
        h = rel2pix(h_rel, img_h)

        rot_rect = ((cx, cy), (w, h), angle)
        box = cv2.boxPoints(rot_rect).astype(np.int32)
        line_y = box[:, 1].max() + 8
        line_x_min = box[:, 0].min()
        line_x_max = box[:, 0].max()

        line_y = max(min(line_y, img_h - edge_gap), edge_gap)
        line_x_min = max(line_x_min, edge_gap)
        line_x_max = min(line_x_max, img_w - edge_gap)

        draw_graffiti_wavy_line(
            img, (line_x_min, line_y), (line_x_max, line_y),
            color=yellow, thickness=wave_thickness, amplitude=wave_amplitude, frequency=22
        )

    return img, th  # 返回th值，供外部使用


# ========== API接口（核心调用入口） ==========
@app.post("/annotate_url")
async def api_annotate_image(req: AnnotationRequest):
    try:
        img = download_image_from_url(req.user_image)
        llm_data = req.llm_output
        if isinstance(llm_data.get("res_text"), str):
            llm_data = json.loads(llm_data["res_text"])

        annotated_img, th = annotate_image(img, llm_data)  # 接收返回的th值

        img_name = f"annotated_{random.randint(1000, 9999)}.png"
        img_path = os.path.join(IMAGE_DIR, img_name)
        cv2.imwrite(img_path, annotated_img, [cv2.IMWRITE_PNG_COMPRESSION, 0])

        img_url = f"http://{SERVER_IP}:8000/static/images/{img_name}"
        return {
            "code": 0,
            "msg": "批注成功",
            "image_url": img_url,
            "base_text_height": th,
            "details": "自适应字高、粗细、位置已全部生效"
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"批注失败：{str(e)}")


# ========== 启动入口 ==========
if __name__ == "__main__":
    import uvicorn

    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 5. 节点5：结束
返回变量：
|变量名|类型|值|
|-|-|:-:|
|output|String|节点4：image_url|
|name|String|节点1：name|

## 三、未来展望
1. 多图批改：设置多图输入、循环OCR识别，串联上下文；
2. 加强健壮性：对于不同偏转角度的作文进行自适应识别；
3. ……