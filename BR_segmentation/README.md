# Breast Segmentation

담당자: 영섭 이
진행 상태: 완료
프로젝트: NIPA (https://www.notion.so/NIPA-28842971c02f80ea8a94f53b5f0fcf47?pvs=21), H&E 레벨 바이오마커 AI (https://www.notion.so/H-E-AI-26142971c02f8019860cefd1752d89d3?pvs=21)
git repositories: https://github.com/Leeyoungsup/NIPA_model

# DataSet

 **NIA 2024 유방암 병리 이미지 및 판독문 합성데이터**

### **소개**

유방암 병리 패치 합성 이미지 및 셀 세그멘테이션, 판독문 라벨링 데이터를 구축함. 유방암 병리 패치 합성 이미지는 정상유방조직, 상피내암, 침윤암으로 구성하여 총 2만 건을 구축하였음.

### **구축목적**

다양한 병리학적 유형, 아형을 포함한 유방암의 전체 슬라이드로부터 추출되는 병리 패치 이미지에 대하여 라벨링된 데이터 및 병리학적 정보를 확보하여 인공지능 학습용 고품질 합성 이미지 및 판독문 데이터를 구축하고자 함.

## 메타데이터 구조표

| **데이터 영역** | 헬스케어 | **데이터 유형** | 이미지 |
| --- | --- | --- | --- |
| **데이터 형식** | PNG | **데이터 출처** | (의료)길의료재단(가천대 길병원), 가톨릭대학교 산학협력단(가톨릭대학교 서울성모병원, 의정부성모병원, 성빈센트병원), 고려대학교 산학협력단(고려대학교 안암병원), 아주대학교 산학협력단(아주대학교 병원) 수집 |
| **라벨링 유형** | 셀 세그멘테이션(이미지), 판독문(자연어) | **라벨링 형식** | JSON |
| **데이터 활용 서비스** | AI 기반 병리 진단 보조 솔루션 | **데이터 구축년도/데이터 구축량** | 2024년/20,000건 |

## 데이터 통계

### 데이터 구축 규모

**○ 유방암 병리 이미지 및 판독문 합성데이터 : 2만 건**

| **데이터명** | **분류** | **라벨링 유형** | **객체수** | **수량** | **단위** |
| --- | --- | --- | --- | --- | --- |
| 유방암 병리 이미지 및 판독문 합성데이터 | 정상유방조직 | Polygon | 11,087,555 | 6,500 | 장 |
|  | 상피내암 | Polygon | 13,229,635 | 6,500 | 장 |
|  | 침윤암 | Polygon | 14,318,015 | 7,000 | 장 |
| 총수량 |  |  |  | 20,000 | 장 |

### 데이터 분포

**○ 유방암 병리 이미지 및 판독문 합성데이터 암종별 분포**

- 정상유방조직 : 6,500장
- 상피내암 : 6,500장
- 침윤암 : 7,000장

# Data Preprocessing

Image : png 형식의 20x 배율 1024x1024 패치이미지 

Label : npy형식의 1024 x 1024 x class 개수 배열 one-hot 구조

```python
#breast
integrated_class={
    "Background":0,
    "NT_stroma": 1,
    "NT_epithelial":2,
    "NT_immune": 3,
    "Tumor": 4,
    "TP_invasive": 5,
    "TP_in_situ": 6,
    
}
image_list=glob('../../data/NIPA/BR*/*.jpeg')
json_list=[f.replace('.jpeg','.json') for f in image_list]
class_count=len(integrated_class)
image_path='../../data/NIPA/breast/images/'
mask_path='../../data/NIPA/breast/masks/'
create_directory(image_path)
create_directory(mask_path)
for i in tqdm(range(len(image_list))):
    image=Image.open(image_list[i])
    width, height=image.size
    json_file=json.load(open(json_list[i]))
    mask=np.zeros((height,width,class_count),dtype=np.uint8)
    for j in range(len(json_file['content']['file']['objects'])):
        label_name=json_file['content']['file']['objects'][j]['label_nm']
        if label_name=="Cell_nucleus":
            continue
        if label_name=="Stroma":
            label_name="NT_stroma"
        channel_index=integrated_class[label_name]
        polygon=json_file['content']['file']['objects'][j]['coordinate']
        temp_mask = mask[:,:,channel_index].copy()
        mask[:,:,channel_index]=polygon2mask(polygon,temp_mask,1)
    mask[:,:,0]=1-(mask[:,:,1]|mask[:,:,2]|mask[:,:,3]|mask[:,:,4]|mask[:,:,5]|mask[:,:,6])
    total_mask = mask.sum(axis=-1) 
    if total_mask.max()>1:
        overlap_indices = np.where(total_mask>1)
        for idx in zip(*overlap_indices):
            # 우선순위: TP_invasive > TP_in_situ > Tumor > NT_epithelial > NT_immune > NT_stroma
            # 가장 높은 우선순위 클래스만 남기고 나머지는 0으로 설정
            
            # TP_invasive가 있으면 다른 모든 클래스 제거
            if mask[idx[0], idx[1], 5] == 1:
                mask[idx[0], idx[1], 1:5] = 0  # NT 계열과 Tumor 모두 제거
                mask[idx[0], idx[1], 6] = 0    # TP_in_situ 제거
                
            # TP_invasive가 없고 TP_in_situ가 있으면
            elif mask[idx[0], idx[1], 6] == 1:
                mask[idx[0], idx[1], 1:5] = 0  # NT 계열과 Tumor 모두 제거
                
            # Tumor 계열이 없고 Tumor가 있으면
            elif mask[idx[0], idx[1], 4] == 1:
                mask[idx[0], idx[1], 1:4] = 0  # NT 계열 모두 제거
                
            # NT_epithelial이 있으면
            elif mask[idx[0], idx[1], 2] == 1:
                mask[idx[0], idx[1], 1] = 0    # NT_stroma 제거
                mask[idx[0], idx[1], 3] = 0    # NT_immune 제거
                
            # NT_immune이 있으면
            elif mask[idx[0], idx[1], 3] == 1:
                mask[idx[0], idx[1], 1] = 0    # NT_stroma 제거
    shutil.copy(image_list[i],image_path+os.path.basename(image_list[i]))
    np.save(mask_path+os.path.basename(image_list[i]).replace('.jpeg','.npy'),mask)
```

# Model

![image.png](image.png)

# Training

![image.png](image%201.png)

![image.png](image%202.png)

![image.png](image%203.png)

# Test Set Performance Evaluation

### Pixel Accuracy:

Mean ± Std: 0.8415 ± 0.1396
95% CI: [0.8355, 0.8474]
Min: 0.0953, Max: 0.9953

### Dice Score per Class:

Overall Mean Dice Score: 0.8254 ± 0.1329

| Class  | Mean  | Std  | 95% CI(Confidence Interval) |
| --- | --- | --- | --- |
| Background | 0.5568 | 0.2500 | [0.5462, 0.5675] |
| NT_stroma | 0.7447 | 0.3248 | [0.7309, 0.7585] |
| NT_epithelial | 0.8555 | 0.2919 | [0.8431, 0.8679] |
| NT_immune | 0.8065 | 0.3555 | [0.7913, 0.8216] |
| Tumor   | 0.9995 | 0.0217 | [0.9986, 1.0005] |
| TP_invasive  | 0.9073 | 0.2145 | [0.8982, 0.9164] |
| TP_in_situ | 0.9076 | 0.2604 | [0.8965, 0.9187] |

# WSI prediction

![image.png](image%204.png)

![image.png](image%203.png)