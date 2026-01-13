# 캐릭터 생성 및 진화 시스템 기술 명세

2.5D 아이소메트릭 게임에서 **랜덤 진화**와 **교배(탄생)** 시스템을 구현하기 위한 기술적 접근 방식과 구현 가이드입니다.

## 1. 핵심 설계: 모듈형 스프라이트 (Modular Sprites)

기존의 프레임 단위 스프라이트(Frame-by-frame) 방식은 모든 진화 조합을 그려낼 수 없으므로, **페이퍼 돌(Paper Doll)** 방식을 채택합니다.

### 1.1 구조 분리 (Parts System)
캐릭터를 단일 이미지가 아닌, 독립적인 파츠(Parts)의 조합으로 구성합니다.
*   **필수 파츠**: 머리(Head), 몸통(Body), 팔(Left/Right Arm), 다리(Left/Right Leg)
*   **추가 파츠**: 눈(Eyes), 입(Mouth), 꼬리(Tail), 장식(Wings, Horns)

### 1.2 Skeleton2D 활용
Godot의 **Skeleton2D** 시스템을 사용하여 파츠가 변경되어도 애니메이션을 공유할 수 있도록 합니다.
1.  **Bone 구조**: 척추, 머리, 팔, 다리 등의 Bone 계층 구조 생성
2.  **BoneAttachment2D / RemoteTransform2D**: 각 Bone 위치에 Sprite2D 연결
3.  **애니메이션 공유**: 텍스처(이미지)만 교체하고, 움직임(AnimationPlayer)은 Bone을 제어하므로 모든 캐릭터가 동일한 애니메이션 데이터 사용 가능

---

## 2. 데이터 구조: DNA 시스템

캐릭터의 외형과 속성을 결정하는 유전 정보(DNA)를 Dictionary 형태로 관리합니다.

```gdscript
# CharacterData.gd 예시
class_name CharacterData

var dna = {
    "species_type": "mammal",      # 종족 타입 (포유류, 파충류 등)
    "head_id": 3,                  # 리소스 ID: res://assets/heads/head_03.png
    "body_id": 1,                  # 리소스 ID: body_01.png
    "color_primary": Color(0.8, 0.2, 0.2), # 주 색상 (RGB)
    "eyes_id": 5,                  # 눈 모양 ID
    "accessory_id": 2,             # 장식 ID (뿔, 안테나 등)
    "scale_modifier": 1.2          # 크기 변수
}
```

---

## 3. 구현 로직

### 3.1 탄생(교배) 및 랜덤 생성 알고리즘
부모의 DNA를 혼합하거나, 돌연변이를 통해 새로운 DNA를 생성합니다.

```gdscript
func generate_offspring(parent_a_dna: Dictionary, parent_b_dna: Dictionary) -> Dictionary:
    var new_dna = {}
    
    # 1. 유전: 50% 확률로 부모 중 하나의 파츠를 물려받음
    new_dna["head_id"] = parent_a_dna["head_id"] if randf() > 0.5 else parent_b_dna["head_id"]
    new_dna["body_id"] = parent_a_dna["body_id"] if randf() > 0.5 else parent_b_dna["body_id"]
    
    # 2. 돌연변이: 10% 확률로 완전히 새로운 파츠 발현
    if randf() < 0.1:
        new_dna["accessory_id"] = randi() % TOTAL_ACCESSORY_COUNT
    else:
        new_dna["accessory_id"] = parent_a_dna["accessory_id"]

    # 3. 색상 혼합: 부모 색상의 중간값 + 랜덤 노이즈(다양성)
    var mix_color = parent_a_dna["color_primary"].lerp(parent_b_dna["color_primary"], 0.5)
    # 색상에 약간의 변이 추가 (생략됨)
    new_dna["color_primary"] = mix_color
    
    return new_dna
```

### 3.2 시각화 (Visualizer)
DNA 정보를 바탕으로 실제 게임 오브젝트의 텍스처와 속성을 변경합니다.

```gdscript
# CharacterVisualizer.gd
extends Node2D

@onready var head_sprite = $Skeleton2D/Hip/Torso/Head/HeadSprite
@onready var body_sprite = $Skeleton2D/Hip/Torso/BodySprite

func apply_dna(dna: Dictionary):
    # 1. 리소스 동적 로드
    var head_path = "res://assets/heads/head_%d.png" % dna["head_id"]
    if ResourceLoader.exists(head_path):
        head_sprite.texture = load(head_path)
    
    # 2. 색상 적용 (Modulate)
    # 전체 톤 변경 또는 셰이더를 통한 특정 색상 변경
    $Skeleton2D.modulate = dna["color_primary"]
    
    # 3. 크기 적용
    scale = Vector2.ONE * dna.get("scale_modifier", 1.0)
```

---

## 4. 아이소메트릭(Isometric) 뷰 고려사항

아이소메트릭 시점에서는 캐릭터의 **방향성** 처리가 중요합니다.

*   **좌우 반전 활용**: `scale.x = -1`을 사용하여 우측면만 작업하고 좌측면은 반전시켜 리소스 절약
*   **리소스 구성**: Bone 구조를 공유하되, 정면(Front-side)과 후면(Back-side) 뷰에 대한 텍스처 세트를 각각 준비하거나, 4방향용 스프라이트 시트를 파츠별로 제작하여 `AtlasTexture`로 관리

## 5. 요약
1.  캐릭터를 **파츠(Parts)** 단위로 분리하여 리소스 제작
2.  **Skeleton2D**로 뼈대 애니메이션 구축 (파츠 교체 가능)
3.  **DNA** 데이터로 외형 정보 저장 및 관리
4.  교배 시 **랜덤/유전 알고리즘**을 통해 DNA 생성 후 `Visualizer`로 적용
