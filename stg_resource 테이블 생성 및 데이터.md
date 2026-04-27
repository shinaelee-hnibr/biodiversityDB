# stg_resource 테이블 생성 및 데이터


## #csv 파일 준비 (주의사항)

- csv 파일을 저장할 때 ktsn 속성을 숫자로 변환하고 저장하기. 숫자가 1.2E+11 이렇게 들어올 수 있음 → 저장하고 항상 확인하기
- csv 파일 저장할때 UTF-8로 저장하기
- 표본 목록 중에서 실적으로 인정된 표본을 계산할 수 있도록 실적연도 컬럼을 만들고 연도 표시를 했음

## #Raw 테이블 생성

CSV의 한글 컬럼명과 구조를 그대로 반영한 테이블 생성

00_create_raw_resource_table.sql

```sql
@set raw_table = s01_raw.raw_specimen_vertebrate_2021_2025_2472

CREATE TABLE ${raw_table} (
    "적재일시"            TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    "번호"                TEXT,
    "과제명"              TEXT,
    "부서명"              TEXT,
    "자원번호"            TEXT,
    "이전관리번호"         TEXT,
    "자원등록일"           TEXT, 
    "자원상태"            TEXT,
    "확증정보 여부"        TEXT,
    "채집지번호"           TEXT,
    "채집일"              TEXT, 
    "이름 (국문)"         TEXT,
    "이름 (영문)"         TEXT,
    "국가명"              TEXT,
    "섬명"                TEXT,
    "채집지 주소 국문"     TEXT,
    "채집지 상세주소 (국문)" TEXT,
    "채집지 주소 (영문)"    TEXT,
    "채집지 상세주소 (영문)" TEXT,
    "위도"                TEXT,
    "경도"                TEXT,
    "고도"                TEXT,
    "위도 - 도"           TEXT, 
    "위도 - 분"           TEXT, 
    "위도 - 초"           TEXT,
    "경도 - 도"           TEXT, 
    "경도 - 분"           TEXT, 
    "경도 - 초"           TEXT, 
    "채집지 비고"          TEXT,
    "학명"                TEXT,
    "국명"                TEXT,
    "자원비고"            TEXT,
    "KTSN"                TEXT,
    "1차분류군"            TEXT,
    "KTSN 문"             TEXT,
    "KTSN 강"             TEXT,
    "KTSN 목"             TEXT,
    "KTSN 과"             TEXT,
    "KTSN 속"             TEXT,
    "KTSN 종"             TEXT,
    "NCBI"                TEXT,
    "GBIF"                TEXT,
    "ITIS"                TEXT,
    "대민 공개여부"        TEXT,
    "대민 비공개사유"      TEXT,
    "대민비공개 사유(기타)" TEXT,
    "비공개기간"           TEXT, 
    "공개기간날짜"         TEXT, 
    "내부 외부여부"        TEXT,
    "신종미기록종여부"      TEXT,
    "임시번호"            TEXT,
    "자원유형"            TEXT,
    "[자원종류] 자료범부"   TEXT,
    "[자원종류] 기준표본 종류" TEXT,
    "Voucher 표본"        TEXT,
    "[분리] 분리숙주세균"   TEXT,
    "[분리] 감염가능숙주세균" TEXT,
    "[분리] 분리일"        TEXT, 
    "[분리] 분리자"        TEXT,
    "[분리] 기능유전자"     TEXT,
    "[동정]동정자"         TEXT,
    "[동정]동정일"         TEXT, 
    "배지이름"            TEXT,
    "[배양/보존] 온도"     TEXT,
    "[배양/보존] 배양방법"  TEXT,
    "염기서열"            TEXT,
    "기증/기탁 여부"       TEXT,
    "기증자"              TEXT,
    "기증기관"            TEXT,
    "분산수장자원여부"      TEXT,
    "분산수장출처기관"      TEXT,
    "분산수장자원번호"      TEXT,
    "등록자"              TEXT,
    "실적연도"            TEXT
);
```

## #csv 파일 업로드

명령어보다 DBeaver의 **'데이터 가져오기(Import Data)'** 기능을 쓰는 것이 훨씬 직관적입니다.

1. DBeaver 왼쪽 트리에서 생성한 specimen_raw_plant_2021_9077 테이블을 **마우스 우클릭**합니다.
2. *데이터 가져오기 (Import Data)**를 선택합니다.
3. CSV 파일을 선택하고 다음(Next)을 누릅니다.
4. Input files: Browse 를 눌러 가져올 파일을 선택합니다.
5. **Mapping** 화면에서 preview data 버튼을 눌러 CSV의 컬럼과 임시 테이블의 컬럼이 잘 연결되었는지 확인합니다.
6. **Start**를 눌러 데이터를 적재합니다.

## #Staging 테이블 생성

**Staging** 테이블은 원본의 "지저분한" 데이터를 마스터로 보내기 전, **가공하는 작업대**입니다.

따라서 단순히 원본 컬럼만 복사하는 게 아니라, **마스터 테이블의 PK와 매칭할 외래키(FK) 후보들**, 그리고 **공간 분석을 위한 Geometry 컬럼** 등을 미리 준비해두는 것이 핵심입니다.

**00_create_str_resource_table.sql**

```sql
-- DBeaver 변수 설정
@set staging_table = s02_staging.stg_specimen_vertebrate_2021_2025_2472

CREATE TABLE ${staging_table} (
    -- [1. 시스템 관리 및 처리 상태]
    staging_id                    BIGSERIAL PRIMARY KEY,
    process_status                TEXT DEFAULT 'WAIT', -- WAIT, DONE, ERROR
    staging_remarks               TEXT,                -- 에러 사유 및 데이터 정제 비고

    -- [2. Project 연계 (공통)]
    project_id                    BIGINT,              -- s03_master.project 연계 ID
    project_name                  TEXT,                -- 원본 과제명

    -- [3. Event(조사) 관련 속성]
    event_id                      BIGINT,              -- s03_master.event 연계 ID
    event_uid                     TEXT,                -- 조사 UID (event_uid)
    event_date                    DATE,                -- 조사일자
    event_year                    INTEGER,             -- 조사연도
    event_type                    TEXT,                -- 조사구분
    location_id                   TEXT,                -- 채집지번호
    location_type                 TEXT,                -- 지역유형
    habitat                       TEXT,                -- 서식지
    sampling_protocol             TEXT,                -- 조사방법
    island_id                     BIGINT,              -- s03_master.island 매칭 결과
    island_name_land              TEXT,                -- 도서명(국토부)
    island_name                   TEXT,                -- IBIS에 입력한 도서명    
    decimal_latitude              NUMERIC(11,8),       -- 위도(10진법)
    decimal_longitude             NUMERIC(12,8),       -- 경도(10진법)
    geodetic_datum                TEXT DEFAULT 'WGS84',
    geom_point_5186               GEOMETRY(Point, 5186),
    verbatim_elevation            TEXT,                -- 현장기록 고도
    verbatim_depth                TEXT,                -- 현장기록 수심
    address_ko                    TEXT,                -- 주소(국문)
    address_detail_ko             TEXT,                -- 상세주소(국문)
    address_en                    TEXT,                -- 주소(영문)
    address_detail_en             TEXT,                -- 상세주소(영문)
    country                       TEXT,                -- 국가명
    country_code                  TEXT,                -- 국가  
    recorded_by_ko                TEXT,                -- 조사자(국문)
    recorded_by_en                TEXT,                -- 조사자(영문)
    event_remarks                 TEXT,                -- 조사비고

    -- [4. Taxon(분류) 및 Occurrence(출현) 관련 속성]
    taxon_id                      BIGINT,              -- s03_master.taxon 연계 ID
    ktsn                          TEXT,                -- KTSN (문자열 보존)
    verbatim_scientific_name      TEXT,                -- 현장학명
    verbatim_common_name          TEXT,                -- 현장국명
    identified_by                 TEXT,                -- 동정자
    identified_date               DATE,                -- 동정일    
    occurrence_id                 BIGINT,              -- s03_master.occurrence 연계 ID
    occurrence_uid                TEXT,                -- 출현 UID (occurrence_uid)
    basis_of_record               TEXT,                -- 기록근거
    individual_count              INTEGER,             -- 개체수
    sex                           TEXT,                -- 성별
    life_stage                    TEXT,                -- 생애주기
    reproductive_condition        TEXT,                -- 번식상태
    is_public                     TEXT,                -- 공개여부
    restriction_reason            TEXT,                -- 비공개사유
    occurrence_remarks            TEXT,                -- 출현비고

    -- [5. Resource(자원/표본) 관련 속성]
    resource_id                   BIGINT,              -- s03_master.resource 연계 ID
    performance_year              INTEGER,             -- 실적연도
    catalog_number                TEXT,                -- 자원번호
    sample_number                 TEXT,                -- 채집번호
    other_catalog_number          TEXT,                -- 분산수장자원번
    preservation_type             TEXT,                -- 자원유형(건조/액침/슬라이드/배양체)
    specimen_category             TEXT,                -- 표본구분(일반표본/기준표본)
    type_status                   TEXT,                -- 기준표본 여부/종류
    sample_quantity               NUMERIC,             -- 표본 개체수
    resource_remarks              TEXT,                -- 자원비고
    
    -- [6. 생성 기록]
    created_at                    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## #Raw → Staging 이관

이 단계에서 데이터를 **'세탁'**하고 **'라벨'**을 붙입니다.

**11_move_raw_to_str_resource_table.sql**

```sql
-- DBeaver 변수 설정
@set raw_table = s01_raw.raw_specimen_vertebrate_2021_2025_2472
@set staging_table = s02_staging.stg_specimen_vertebrate_2021_2025_2472

INSERT INTO ${staging_table} (
    -- 프로젝트 정보
    project_name,
    
    -- Event (조사 정보)
    event_date,
    event_year,
    location_id, 
    island_name,        --ibis에 기재된 섬명 
    decimal_latitude, 
    decimal_longitude, 
    verbatim_elevation, 
    address_ko, 
    address_detail_ko, 
    address_en, 
    address_detail_en,
    country, 
    recorded_by_ko, 
    recorded_by_en, 
    event_remarks,
    geom_point_5186,
    
    -- Taxon & Occurrence (분류 및 출현 정보)
    ktsn, 
    verbatim_scientific_name, 
    verbatim_common_name,
    identified_by, 
    identified_date, 
    is_public, 
    restriction_reason, 
    occurrence_remarks,
    
    -- Resource (표본 정보)
    performance_year,
    catalog_number,
    other_catalog_number, 
    preservation_type,
    specimen_category, 
    type_status
)
SELECT 
    -- [프로젝트]
    NULLIF(TRIM("과제명"), ''),
    
    -- [이벤트] 텍스트 날짜를 DATE 타입으로 안전하게 변환
    NULLIF(TRIM("채집일"), '')::DATE,
    EXTRACT(YEAR FROM NULLIF(TRIM("채집일"), '')::DATE)::INTEGER, -- 연도 자동 추출
    NULLIF(TRIM("채집지번호"), ''), 
    NULLIF(TRIM("섬명"), ''), 
    NULLIF(TRIM("위도"), '')::NUMERIC, 
    NULLIF(TRIM("경도"), '')::NUMERIC, 
    NULLIF(TRIM("고도"), ''), 
    NULLIF(TRIM("채집지 주소 국문"), ''), -- raw 테이블 컬럼명에 맞춰 수정
    NULLIF(TRIM("채집지 상세주소 (국문)"), ''), 
    NULLIF(TRIM("채집지 주소 (영문)"), ''), 
    NULLIF(TRIM("채집지 상세주소 (영문)"), ''),
    NULLIF(TRIM("국가명"), ''), -- raw 테이블의 '국가명' 매핑
    NULLIF(TRIM("이름 (국문)"), ''), -- 채집자 국문
    NULLIF(TRIM("이름 (영문)"), ''), -- 채집자 영문
    NULLIF(TRIM("채집지 비고"), ''),
    
    -- [공간 데이터 변환] TEXT 위경도를 숫자로 변환 후 Point 생성
    CASE 
        WHEN NULLIF(TRIM("위도"), '') IS NOT NULL AND NULLIF(TRIM("경도"), '') IS NOT NULL 
        THEN ST_Transform(ST_SetSRID(ST_MakePoint(NULLIF(TRIM("경도"), '')::NUMERIC, NULLIF(TRIM("위도"), '')::NUMERIC), 4326), 5186)
        ELSE NULL 
    END,
    
    -- [분류 및 출현]
    NULLIF(TRIM("KTSN"), ''), 
    NULLIF(TRIM("학명"), ''), 
    NULLIF(TRIM("국명"), ''),
    NULLIF(TRIM("[동정]동정자"), ''), 
    NULLIF(TRIM("[동정]동정일"), '')::DATE, 
    NULLIF(TRIM("대민 공개여부"), ''), 
    NULLIF(TRIM("대민 비공개사유"), ''), 
    NULLIF(TRIM("자원비고"), ''), 
    
    -- [자원]
    NULLIF(TRIM("실적연도"), '')::INTEGER, 
    NULLIF(TRIM("자원번호"), ''),
    NULLIF(TRIM("분산수장자원번호"), ''),
    NULLIF(TRIM("자원유형"), ''),
    NULLIF(TRIM("[자원종류] 자료범부"), ''),  
    NULLIF(TRIM("[자원종류] 기준표본 종류"), '')

FROM ${raw_table};
```

### 전체 이관이 잘 되었는지 검증

raw 테이블과 stagin 테이블 행개수를 비교하여 전체 데이터가 문제없이 잘 들어 갔는지 확인 

```sql
@set raw_table = s01_raw.raw_specimen_vertebrate_2021_2025_2472
@set staging_table = s02_staging.stg_specimen_vertebrate_2021_2025_2472

SELECT 
    (SELECT COUNT(*) FROM ${raw_table}) AS raw_count,
    (SELECT COUNT(*) FROM ${staging_table}) AS staging_count,
    (SELECT COUNT(*) FROM ${raw_table}) - (SELECT COUNT(*) FROM ${staging_table}) AS difference;
```
