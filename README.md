# SimpleFS: Design and Implementation of a Minimal File System

### 파일시스템 구조 설계 및 구현 보고서

단국대학교 모바일시스템공학과
2025년 12월 25일

---

# 목차

* [1. 프로젝트 개요](#1-프로젝트-개요)

  * [1.1 프로젝트 목적](#11-프로젝트-목적)
* [2. Execution Results and Validation](#2-execution-results-and-validation)

  * [2.1 Filesystem Mount State](#21-filesystem-mount-state)
  * [2.2 Root Directory Listing](#22-root-directory-listing)
  * [2.3 File Read Operations](#23-file-read-operations)
  * [2.4 Buffer Cache Statistics](#24-buffer-cache-statistics)
  * [2.5 Discussion](#25-discussion)
  * [2.6 Summary](#26-summary)
* [3. 파일 시스템 구조 및 설계 선택](#3-파일-시스템-구조-및-설계-선택)

  * [3.1 디스크 레이아웃 및 전체 구조](#31-디스크-레이아웃-및-전체-구조)
  * [3.2 Superblock 및 비트맵 기반 자원 관리](#32-superblock-및-비트맵-기반-자원-관리)

    * [3.2.1 Superblock 구조](#321-superblock-구조)
    * [3.2.2 비트맵 저장 방식 및 코드 반영](#322-비트맵-저장-방식-및-코드-반영)
    * [3.2.3 과제 요구 기능과의 연관성](#323-과제-요구-기능과의-연관성)
    * [3.2.4 설계 선택 이유](#324-설계-선택-이유)
  * [3.3 Inode 구조 및 블록 매핑 방식](#33-inode-구조-및-블록-매핑-방식)

    * [3.3.1 Inode 구조](#331-inode-구조)
    * [3.3.2 블록 매핑 방식](#332-블록-매핑-방식)
    * [3.3.3 제약 사항 및 불완전성](#333-제약-사항-및-불완전성)
  * [3.4 추가 개선 가능 사항 요약 (설계 관점)](#34-추가-개선-가능-사항-요약-설계-관점)

    * [3.4.1 자원 할당 탐색 비용 완화](#341-자원-할당-탐색-비용-완화)
    * [3.4.2 메타데이터 집중 구조 완화](#342-메타데이터-집중-구조-완화)
    * [3.4.3 쓰기 연산 안정성 개선](#343-쓰기-연산-안정성-개선)
  * [3.5 Root File System Mount](#35-root-file-system-mount)

    * [3.5.1 FS 구조체 전달 방식과 설계 의도](#351-fs-구조체-전달-방식과-설계-의도)
    * [3.5.2 Superblock 중심 상태 초기화](#352-superblock-중심-상태-초기화)
    * [3.5.3 확장성을 고려한 상태 전달 구조](#353-확장성을-고려한-상태-전달-구조)
    * [3.5.4 Directory Hash Cache를 mount 시점에 구축한 이유](#354-directory-hash-cache를-mount-시점에-구축한-이유)
    * [3.5.5 Directory Hash Cache의 mount 시점 구축](#355-directory-hash-cache의-mount-시점-구축)
* [4. 파일 연산 흐름과 상태 전이 기반 구조 분석](#4-파일-연산-흐름과-상태-전이-기반-구조-분석)

  * [4.1 File Operation 흐름 (open / read / write / close)](#41-file-operation-흐름-open--read--write--close)

    * [4.1.1 open 연산 흐름](#411-open-연산-흐름)
    * [4.1.2 read 연산 흐름](#412-read-연산-흐름)
    * [4.1.3 write 연산 흐름](#413-write-연산-흐름)
    * [4.1.4 close 연산 흐름](#414-close-연산-흐름)
    * [4.1.5 연산 흐름 요약](#415-연산-흐름-요약)
  * [4.2 File Operation 흐름에 대한 구조적 문제 분석](#42-file-operation-흐름에-대한-구조적-문제-분석)

    * [4.2.1 문제 요약: 중간 상태가 외부에 노출될 수 있는 구조](#421-문제-요약-중간-상태가-외부에-노출될-수-있는-구조)
    * [4.2.2 open 경로에서 발생할 수 있는 비용 집중 문제](#422-open-경로에서-발생할-수-있는-비용-집중-문제)
    * [4.2.3 read/write 경로에서의 처리 분리](#423-readwrite-경로에서의-처리-분리)
    * [4.2.4 write 연산에서 발생하는 부분 반영 문제](#424-write-연산에서-발생하는-부분-반영-문제)
    * [4.2.5 왜 journaling과 WAL이 필요한가](#425-왜-journaling과-wal이-필요한가)
    * [4.2.6 요약](#426-요약)
  * [4.3 Flash 연산 제약이 파일 시스템 구조에 반영되는 방식](#43-flash-연산-제약이-파일-시스템-구조에-반영되는-방식)

    * [4.3.1 Flash memory의 기본 연산 모델](#431-flash-memory의-기본-연산-모델)
    * [4.3.2 중간 상태가 허용되지 않는 이유](#432-중간-상태가-허용되지-않는-이유)
    * [4.3.3 Flash 제약이 반영된 littlefs 구조](#433-flash-제약이-반영된-littlefs-구조)
    * [4.3.4 구조적 차이의 필연성](#434-구조적-차이의-필연성)
* [5. 결론](#5-결론)
* [부록 A. Execution Log Excerpt](#부록-a-execution-log-excerpt)

---

# 1. 프로젝트 개요

## 1.1 프로젝트 목적

* 디스크 이미지 기반의 간단한 파일 시스템(SimpleFS)을 설계하고 구현했습니다. 이를 통해 운영체제에서 파일 시스템이 수행하는 메타데이터 관리, 데이터 접근, 상태 전이 과정을 코드 수준에서 직접 확인하는 것을 목표로 했습니다.

* 파일 시스템은 Superblock, inode, directory entry와 같은 핵심 자료구조를 중심으로 설계했으며, 파일 이름이 디렉터리를 통해 inode로 연결되고 실제 데이터 블록에 매핑되는 구조를 구현했습니다. 또한 open, read, write, close 연산을 직접 구현하여 파일 시스템의 기본 동작을 검증했습니다.

* 추가적으로 디렉터리 해시 캐시와 buffer cache를 도입하여 파일 접근 성능을 개선했으며, 데이터 블록 할당자와 디스크 이미지 생성 도구를 분리 구현하여 구조적인 완성도를 높였습니다. 이 과정을 통해 파일 시스템의 내부 동작과 설계 요소를 보다 명확히 이해할 수 있었습니다.

---

# 2. Execution Results and Validation

본 절에서는 SimpleFS 구현의 실행 결과를 제시하고,
파일 시스템의 마운트, 디렉터리 파싱, 파일 읽기 동작이
의도한 설계에 따라 정상적으로 수행됨을 검증합니다.

## 2.1 Filesystem Mount State

`fs_mount()` 실행 결과,
파일 시스템 메타데이터가 정상적으로 파싱되었음을 확인합니다.

* 루트 inode 번호는 **inode 2**로 설정되어 있으며,
  superblock의 `first_inode` 필드와 일치합니다.
* 루트 디렉터리의 크기는 **3264 bytes**로,
  디렉터리 엔트리들의 총 크기와 합리적으로 일치합니다.
* 루트 디렉터리는 간접 블록을 사용하지 않으며,
  4개의 직접 데이터 블록만을 사용합니다.

이를 통해 superblock, inode 테이블, 데이터 블록 매핑 정보가
일관된 상태로 로드되었음을 확인합니다.

## 2.2 Root Directory Listing

마운트 직후 루트 디렉터리 목록을 출력한 결과는 다음과 같습니다.

* `.` 및 `..` 엔트리가 정상적으로 존재합니다.
* `file_1`부터 `file_92`까지의 파일이
  중복 없이 한 번씩만 출력됩니다.
* inode 번호는 연속적으로 증가하며,
  디렉터리 파싱 과정에서 무한 반복이나
  잘못된 엔트리 해석은 발생하지 않습니다.

이를 통해 루트 디렉터리 파싱 로직과
디렉터리 해시 캐시가 정상적으로 동작함을 확인합니다.

## 2.3 File Read Operations

무작위로 선택된 여러 파일에 대해
`open()` 및 `read()`를 수행한 결과는 다음과 같습니다.

* 각 파일은 정상적으로 열립니다.
* 요청한 바이트 수만큼의 데이터가 반환됩니다.
* 동일 파일을 반복하여 읽을 경우
  항상 동일한 내용이 반환됩니다.

이는 inode에서 데이터 블록으로의 매핑이 올바르며,
읽기 경로에서 메모리 손상이나
잘못된 블록 접근이 발생하지 않음을 의미합니다.

## 2.4 Buffer Cache Statistics

실행 종료 시 출력된 버퍼 캐시 통계를 통해
다음과 같은 특성을 확인합니다.

* 초기 접근 시 일부 cache miss가 발생합니다.
* 이후 반복 접근에서는 높은 cache hit 비율을 보입니다.
* 쓰기 연산이 없으므로 writeback 횟수는 0입니다.
* eviction 횟수는 LRU 정책에 따라 합리적인 수준을 유지합니다.

이를 통해 버퍼 캐시가 정상적으로 초기화되고,
의도한 정책에 따라 동작함을 확인합니다.

## 2.5 Discussion

본 실행 결과는 SimpleFS 구현이
파일 시스템의 핵심 구조적 불변식(invariant)을
정상적으로 만족하고 있음을 보여줍니다.

1. 루트 inode가 superblock 메타데이터와
   일관되게 해석되며,
   모든 디렉터리 및 파일 접근이
   해당 inode를 기준으로 수행됩니다.
2. 루트 디렉터리 파싱 과정에서
   중복 출력이나 무한 반복이 발생하지 않으며,
   각 디렉터리 엔트리가 정확히 한 번씩만 처리됩니다.
3. 파일 읽기 결과가 파일별로 일관되게 유지되며,
   버퍼 캐시를 포함한 읽기 경로 전반에서
   데이터 무결성이 보장됩니다.

이를 통해 본 구현이 읽기 전용 파일 시스템으로서
안정적인 상태 전이를 수행함을 확인합니다.

## 2.6 Summary

위 실행 결과를 통해 SimpleFS의 마운트 과정,
루트 디렉터리 파싱, 파일 읽기 경로가
정상적으로 구현되었음을 확인합니다.

현재 구현은 읽기 전용 파일 시스템으로서
일관된 상태와 안정적인 동작을 보장합니다.

실제 실행 로그의 일부는
부록 [Execution Log Excerpt](#부록-a-execution-log-excerpt)에 제시합니다.

---

# 3. 파일 시스템 구조 및 설계 선택

SimpleFS의 전체 구조와 각 구성 요소의 설계 의도입니다.
비트맵 기반 자원 관리와 캐시 구조를 중심으로 파일 시스템의 설계 방향을 정리했습니다.

## 3.1 디스크 레이아웃 및 전체 구조

```c
typedef struct {
    FILE* disk;
    struct super_block sb;
    struct inode inode_table[MAX_INODES];

    // allocator bitmaps stored in sb.padding
    uint8_t* inode_bm;   // bit i => inode i in use
    uint8_t* block_bm;   // bit bi => data block bi in use
    uint32_t data_blocks; // number of data blocks (derived)
    uint32_t inode_count;

    // caches
    DirHash  dircache;
    BufCache bcache;
} FS;
```

* 디스크 이미지는 고정 크기의 블록 단위로 구성되며,
  Block 0은 superblock, Block 1부터 Block 7까지는 inode table,
  Block 8 이후는 data block 영역으로 배치합니다.
* 파일 시스템의 전반적인 상태는 `FS` 구조체에서 관리하며,
  해당 구조체는 디스크 포인터, superblock, inode table, 자원 할당을 위한 비트맵,
  그리고 성능 향상을 위한 캐시 구조를 포함합니다.

## 3.2 Superblock 및 비트맵 기반 자원 관리

### 3.2.1 Superblock 구조

```c
struct super_block {
    uint32_t partition_type;
    uint32_t block_size;
    uint32_t inode_size;
    uint32_t first_inode;
    uint32_t num_inodes;
    uint32_t num_inode_blocks;
    uint32_t num_free_inodes;
    uint32_t num_blocks;
    uint32_t num_free_blocks;
    uint32_t first_data_block;
    char     volume_name[24];
    uint8_t  padding[960];
};
```

* superblock(`struct super_block`)은 파일 시스템의 전역 메타데이터를 관리합니다.
* 블록 크기, inode 개수, free inode 및 free block 개수, 데이터 블록 시작 위치 등
  파일 시스템 전반의 상태 정보가 포함됩니다.

### 3.2.2 비트맵 저장 방식 및 코드 반영

* 본 구현에서는 **inode bitmap과 data block bitmap을 superblock 내부의 `padding[960]` 영역에 저장**했습니다.
* 별도의 비트맵 블록을 두지 않고 여유 공간을 활용함으로써, 파일 시스템의 **자원 할당 상태를 단일 블록에서 관리**하도록 설계했습니다.

1. 비트맵들은 파일 시스템 mount 과정에서
   `fs_mount()` 함수에 의해 메모리로 로드됩니다.
2. 이후 `FS` 구조체의 `inode_bm`과
   `block_bm` 포인터를 통해 직접 참조됩니다.
3. 추가적인 디스크 접근 없이 inode와 데이터 블록의
   사용 여부를 즉시 확인할 수 있습니다.

### 3.2.3 과제 요구 기능과의 연관성

이러한 설계는 과제 명세에서 요구하는 기본 파일 시스템 기능과
직접적으로 연결됩니다.

* **open 연산**에서는 디렉토리 탐색을 통해 얻은 inode 번호가
  이미 사용 중인지 여부를 inode bitmap을 통해 확인합니다.
* 또한 **write 연산**에서는 block bitmap을 이용해 새로운 데이터 블록을 할당하고, inode의 블록 포인터를 갱신하여 파일 데이터를 저장합니다.

| **구성 요소**                | **역할**             | **설계 선택 / trade-off**           |
| ------------------------ | ------------------ | ------------------------------- |
| **Superblock**           | 파일 시스템 전역 정보 관리    | 비트맵 포함 단일 블록 관리 → 단순함 / 손상 시 취약 |
| **Inode Table**          | 파일 메타데이터 및 블록 위치   | Direct 중심 매핑 → 빠름 / 크기 제한       |
| **Directory Entry**      | 파일 이름 → inode 매핑   | 순차 탐색 구조 → 단순 / 확장성 제한          |
| **Inode / Block Bitmap** | inode, block 사용 여부 | 비트 연산 기반 관리 → 단순 / 선형 탐색        |
| **Directory Hash Cache** | 이름 → inode 캐시      | open 속도 향상 → 캐시 관리 필요           |
| **Buffer Cache**         | 디스크 블록 캐시          | I/O 감소 → 동기화 필요                 |
| **FS 구조체**               | 전체 상태 통합 관리        | 중앙 집중 구조 → 구조 명확                |

**표: SimpleFS 구성 요소 요약**

### 3.2.4 설계 선택 이유

```c
/* Block allocator (bitmap-based) */
static int alloc_dblk(FS* fs, uint32_t* out_bi) {
    for (uint32_t bi = 0; bi < fs->data_blocks; bi++) {
        if (!bm_test(fs->block_bm, bi)) {
            bm_set(fs->block_bm, bi);
            fs->sb.num_free_blocks--;
            *out_bi = bi;
            return 0;
        }
    }
    return -1;
}

static void free_dblk(FS* fs, uint32_t bi) {
    if (bm_test(fs->block_bm, bi)) {
        bm_clear(fs->block_bm, bi);
        fs->sb.num_free_blocks++;
    }
}
```

비트맵 방식을 선택한 이유는 다음과 같습니다.

1. inode와 데이터 블록의 사용 여부를 비트 연산을 통해
   **상수 시간에 확인**할 수 있어,
   자원 할당 및 해제 로직을 단순하게 구현할 수 있습니다.
2. free list 방식에 비해 메타데이터 크기가 작아
   superblock 내부에 충분히 수용 가능하며,
   추가적인 관리 구조가 필요하지 않습니다.
3. inode와 데이터 블록의 할당 및 해제 과정이
   `bm_test`, `bm_set`, `bm_clear`와 같은
   비트 연산 함수 호출로 명확히 표현되어,
   `alloc_inode()`, `alloc_dblk()`,
   `free_dblk()`와 같은 자원 관리 로직이
   간결하고 직관적인 형태로 구현됩니다.

반면, 비트맵 기반 할당은 trade-off를 가집니다.

* free bit를 탐색하기 위해 선형 탐색이 필요하므로, free 자원이 줄어들수록 할당 비용이 증가합니다.
* 또한 비트맵을 superblock 내부에 포함할 경우, superblock 손상 시 파일 시스템 전체의 복구 가능성이 낮아질 수 있습니다.

## 3.3 Inode 구조 및 블록 매핑 방식

### 3.3.1 Inode 구조

```c
struct inode {
    uint32_t mode;           // file type/perm (course-defined)
    uint32_t locked;         // opened for write (optional)
    uint32_t date;
    uint32_t size;           // bytes
    int32_t  indirect_block; // -1 if none; else data block index
    uint16_t blocks[6];      // 6 direct data block indices
};
```

* 파일의 메타데이터와
  데이터 블록 매핑 정보를 관리합니다.
* 파일의 크기, 타입 등의 메타데이터와 함께,
  데이터 블록의 위치를 저장합니다.

### 3.3.2 블록 매핑 방식

#### 파일 크기가 작은 경우

**`blocks[6]` 배열을 이용한 direct block 방식**을
기본으로 사용합니다.
이를 통해 크기가 작은 파일에 대해서는
추가적인 간접 참조 없이 데이터 블록에 바로 접근할 수 있도록 설계했습니다.

#### 파일 크기가 증가하는 경우

파일 크기가 증가하는 경우에는
**`indirect_block` 필드를 사용하여**
추가 데이터 블록을 간접적으로 참조하도록 했습니다.
이를 통해 단순한 구조를 유지하면서도
일정 수준의 파일 크기 확장을 지원합니다.

### 3.3.3 제약 사항 및 불완전성

비트맵 기반 자원 관리는 구조가 단순하고 구현이 명확하다는 장점이 있으나,
몇 가지 제약과 불완전성을 함께 가집니다.

* 우선 free bit를 탐색하는 과정이 **선형 탐색**으로 구현되어 있어,
* 사용 가능한 inode나 데이터 블록의 수가 감소할수록
* 자원 할당에 필요한 탐색 비용이 증가할 수 있습니다.

이로 인해 대규모 파일 시스템이나
높은 할당 빈도를 가지는 환경에서는
자원 할당 시 탐색 비용 증가로 인한
성능 저하가 발생할 수 있습니다.

이러한 제약은 다음과 같은 자원 할당 코드에서 직접적으로 드러납니다.

```c
/* Bitmap-based data block allocator */
static int alloc_dblk(FS* fs, uint32_t* out_bi) {
    for (uint32_t bi = 0; bi < fs->data_blocks; bi++) {
        if (!bm_test(fs->block_bm, bi)) {
            bm_set(fs->block_bm, bi);
            fs->sb.num_free_blocks--;
            *out_bi = bi;
            return 0;
        }
    }
    return -1;
}
```

위 코드에서는 항상 첫 번째 블록부터 순차적으로 탐색을 수행하므로,
free block이 줄어들수록 탐색 범위가 커지는 구조적 한계를 가집니다.
또한 inode 및 데이터 블록 비트맵을
superblock 내부에 함께 저장함으로써,
**superblock이 손상될 경우 inode와 데이터 블록의
할당 정보를 동시에 복구하기 어렵다**는
메타데이터 안정성 측면의 trade-off가 존재합니다.

실제 파일 시스템에서는 이러한 문제를 완화하기 위해
비트맵을 여러 블록에 **분산 저장**하거나,
**저널링 기법**을 도입하여 메타데이터의 일관성과
복구 가능성을 보장합니다.

## 3.4 추가 개선 가능 사항 요약 (설계 관점)

앞서 언급한 제약 사항을 바탕으로,
본 파일 시스템 설계는 다음과 같은 방향으로
확장 및 개선될 여지를 가집니다.
이 절에서는 구체적인 구현보다는,
**현재 구조를 유지한 상태에서 고려할 수 있는 설계 아이디어**를
중심으로 정리합니다.

### 3.4.1 자원 할당 탐색 비용 완화

현재 비트맵 기반 자원 할당은
항상 비트맵의 시작 위치부터 **선형 탐색**을 수행합니다.
이로 인해 free 자원이 줄어들수록
할당 비용이 증가하는 구조적 한계를 가집니다.

이를 완화하기 위한 설계 아이디어로는 다음과 같은 방법을 고려할 수 있습니다.

* 최근에 할당된 위치 이후부터 탐색을 시작하는 방식
* 자주 사용되는 영역과 그렇지 않은 영역을
  논리적으로 분리하여 관리하는 방식

이러한 접근은 전체 자료구조를 변경하지 않으면서도,
**평균적인 자원 할당 비용을 줄이는 데 기여**할 수 있습니다.

**자원 할당 탐색 개선 아이디어**

```c
static uint32_t alloc_hint = 0;

for (uint32_t i = 0; i < fs->data_blocks; i++) {
    uint32_t idx = (alloc_hint + i) % fs->data_blocks;
    if (!bm_test(fs->block_bm, idx)) {
        bm_set(fs->block_bm, idx);
        alloc_hint = idx + 1;
        break;
    }
}
```

### 3.4.2 메타데이터 집중 구조 완화

현재 설계에서는 inode 및 데이터 블록 비트맵이
**superblock 내부에 집중되어 관리**됩니다.
이로 인해 단일 메타데이터 손상 시
파일 시스템 전체에 영향을 줄 수 있는 한계가 존재합니다.

이를 완화하기 위해 다음과 같은 설계 아이디어를 고려할 수 있습니다.

* 비트맵 정보를 별도의 메타데이터 영역으로 분리하는 방식
* 비트맵의 읽기 전용 복사본을 함께 유지하는 방식

이와 같은 구조는
**메타데이터 손상으로 인한 전체 장애 가능성을 줄일 수 있습니다.**

**메타데이터 안정성 향상 아이디어**

```c
/* keep an auxiliary copy for consistency checking */
uint8_t* inode_bm_backup;
uint8_t* block_bm_backup;
```

### 3.4.3 쓰기 연산 안정성 개선

현재 write 연산은 데이터와 메타데이터를
즉시 갱신하는 단순한 흐름으로 구현되어 있습니다.
이로 인해 비정상 종료 상황에서는
일관성 문제가 발생할 수 있습니다.

향후 확장에서는 다음과 같은
**단계적 처리 흐름**를 고려할 수 있습니다.

* 쓰기 연산의 의도를 먼저 기록하고
* 이후 실제 데이터 및 메타데이터 변경을 적용하는 방식

이러한 접근은 write 경로를 명확히 분리하고,
**상태 전이를 보다 명시적으로 관리**할 수 있게 합니다.

**쓰기 연산 안정성 개선 아이디어**

```c
/* record intended update */
log_write_intent(inode_no, target_block);

/* apply actual update */
apply_block_update(...);
```

다만 본 프로젝트에서는 과제 명세에서 요구한
**파일 시스템의 핵심 동작 흐름과 자료구조의 이해**에 초점을 맞추어,
위와 같은 개선 사항을 실제 구현으로 확장하지는 않았습니다.
해당 항목들은 현재 구조를 기반으로 한
**설계 수준의 확장 가능성**으로 정리합니다.

## 3.5 Root File System Mount

본 프로젝트에서는 프로그램 시작 시
root file system이 자동으로 mount되도록 구현했습니다.
mount 과정은 전역 상태를 한 번에 초기화하고,
이후 수행되는 모든 파일 연산이
동일한 기준 상태를 공유하도록 만드는 단계입니다.

이를 위해 파일 시스템의 모든 상태는
`FS` 구조체 하나로 통합 관리되며,
mount 과정에서는 이 구조체를 초기화하고
디스크와 메모리 상태를 연결합니다.

### 3.5.1 FS 구조체 전달 방식과 설계 의도

파일 시스템 mount는
다음과 같이 `FS` 구조체의 주소를 전달하는 방식으로
수행됩니다.

**파일 시스템 인스턴스 생성 및 mount 호출**

```c
FS fs;

if (fs_mount(&fs, argv[1]) != 0)
    die("fs_mount failed");
```

#### **파일 시스템 상태의 캡슐화**

`fs_mount()` 함수는 파일 시스템의 모든 전역 상태를
단일한 **`FS` 구조체**로 캡슐화하고,
이후의 모든 파일 시스템 연산이
해당 구조체의 포인터를 통해
**동일한 상태 집합**을 참조하도록
파일 시스템을 초기화합니다.
`FS` 구조체는
파일 시스템 인스턴스를 대표하는
**유일한 상태 객체**로 동작합니다.

#### **다중 인스턴스 및 테스트 활용성**

전역 상태에 의존하지 않는 구조는
다중 파일 시스템 인스턴스를
동시에 유지하는 것을 가능하게 하며,
테스트 환경에서도
서로 독립적인 파일 시스템 상태를
손쉽게 구성할 수 있게 합니다.

#### **API 관점에서의 의존성 역전**

모든 파일 시스템 연산이
`FS*`를 인자로 받도록 설계함으로써,
상위 수준의 연산 로직은
전역 상태나 숨겨진 구현 세부 사항이 아니라,
**명시적으로 전달된 상태 객체**에만
의존하도록 합니다.
이는 파일 시스템 API가
**의존성 역전 원칙**을
자연스럽게 만족하도록 만듭니다.

### 3.5.2 Superblock 중심 상태 초기화

`fs_mount()` 내부에서는
전달받은 `FS` 구조체를 기준으로
superblock을 디스크에서 읽어옵니다.
이 과정에서 초기화된 superblock은
해당 파일 시스템 인스턴스의 기준 정보로 사용됩니다.

**superblock을 FS 구조체에 적재**

```c
fs->disk = fopen(disk_img_path, "rb+");
if (!fs->disk) return -1;

if (fread(&fs->sb, 1, sizeof(fs->sb), fs->disk)
    != sizeof(fs->sb)) {
    return -1;
}
```

이후 inode 개수와 데이터 블록 개수는
superblock의 정보를 바탕으로 계산되며,
이 값들은 `FS` 구조체 내부에 저장되어
파일 연산 전반에서 재사용됩니다.

**superblock 기반 파생 정보 계산**

```c
fs->inode_count = fs->sb.num_inodes;
fs->data_blocks =
    fs->sb.num_blocks - fs->sb.first_data_block;
```

이와 같은 구조를 통해,
파일 시스템의 핵심 메타데이터는
항상 `FS` 구조체를 통해 접근되며,
open, read, write, close와 같은 연산에서도
동일한 기준 상태를 공유하게 됩니다.

### 3.5.3 확장성을 고려한 상태 전달 구조

`FS` 구조체를 포인터로 전달하는 방식은
확장성 측면에서도 의미를 가집니다.
동일한 인터페이스를 유지한 채,
여러 개의 파일 시스템 인스턴스를
동시에 관리할 수 있기 때문입니다.

**파일 연산에서 FS 인스턴스 명시적 사용**

```c
int sys_open(PCB* pcb, FS* fs, const char* pathname, int flags);
int sys_read(PCB* pcb, FS* fs, int fd, void* buf, uint32_t size);
```

각 파일 연산은
전역 상태가 아닌

* 명시적으로 전달된 `FS` 인스턴스를 기준으로 수행되며,
* 이는 Linux 커널에서 각 마운트된 파일 시스템이
* 자신만의 `struct super_block`을 가지는 구조와 개념적으로 대응됩니다.

이러한 설계를 통해,
mount 과정은 단순한 초기화 단계를 넘어
파일 시스템 인스턴스의 수명과 범위를 정의하는
핵심 단계로 기능하도록 구성했습니다.

### 3.5.4 Directory Hash Cache를 mount 시점에 구축한 이유

본 프로젝트에서는
디렉토리 해시 캐시(`DirHash`)를
파일 시스템 mount 과정에서 미리 구축하도록 설계했습니다.

* 이는 open 연산의 수행 경로를 단순화하고,
* 파일 이름 탐색 비용을 줄이기 위한 선택입니다.

만약 open 호출마다 디렉토리 파일을 디스크에서 읽고
directory entry를 순차 탐색한다면,
파일 개수가 증가할수록 open 연산의 비용이 선형적으로 증가하게 됩니다.

이를 방지하기 위해,

* **mount 시점에 루트 디렉토리의 directory entry를 한 번만 순회**하고,
* 파일 이름과 inode 번호의 매핑을 메모리 상의 해시 구조로 캐시했습니다.

### 3.5.5 Directory Hash Cache의 mount 시점 구축

본 프로젝트에서는 디렉토리 해시 캐시를
파일 시스템 mount 과정에서 한 번만 구축하도록 설계했습니다.
이를 통해 이후의 **open 연산에서
디렉토리 파일을 반복적으로 순회하지 않도록** 했습니다.

mount 과정의 마지막 단계에서는

* 루트 디렉토리 inode를 기준으로directory entry를 순회하며
* 파일 이름과 inode 번호의 매핑을 메모리 상의 해시 구조에 저장합니다.

**mount 시 디렉토리 해시 캐시 구축 및 사용 흐름**

```c
if (parse_root_and_build_cache(fs) != 0)
    return -1;

dirhash_put(&fs->dircache,
            (const char*)de.name,
            de.name_len,
            de.inode);

if (dirhash_get(&fs->dircache, name, len, &ino) != 0)
    return -1;
```

이와 같은 구조를 통해
디렉토리 파일은 mount 시점에 한 번만 읽히며,
**open 연산에서는
디스크 접근 없이
파일 이름을 inode 번호로 즉시 변환**할 수 있습니다.
즉, open 연산의 핵심 경로는
**디렉토리 파일 순회**가 아닌
**해시 조회**로 단순화됩니다.

---

# 4. 파일 연산 흐름과 상태 전이 기반 구조 분석

## 4.1 File Operation 흐름 (open / read / write / close)

본 절에서는 앞서 mount 과정에서 초기화된
파일 시스템 구조가
open, read, write, close 연산에서
어떻게 그대로 활용되는지를
연산 흐름 중심으로 설명합니다.
각 시스템콜은
superblock, inode table, 디렉토리 캐시, 버퍼 캐시를
단계적으로 참조하며 동작하도록 구현되었습니다.

### 4.1.1 open 연산 흐름

open 연산은
파일 이름을 inode 번호로 변환하고,
해당 inode를 기준으로
파일 접근 상태를 생성하는 단계입니다.
본 구현에서는 mount 시점에 구축된
디렉토리 해시 캐시를 통해
파일 이름 탐색을 수행합니다.

**open 연산의 핵심 흐름**

```c
int sys_open(PCB* pcb, FS* fs, const char* pathname, int flags) {
    const char* name = pathname + 1;
    uint32_t len = (uint32_t)strnlen(name, 255);

    uint32_t ino = 0;
    if (dirhash_get(&fs->dircache, name, len, &ino) != 0)
        return -1;

    for (int fd = 0; fd < MAX_FD; fd++) {
        if (!pcb->fdtable[fd].used) {
            pcb->fdtable[fd].used = 1;
            pcb->fdtable[fd].inode_no = ino;
            pcb->fdtable[fd].offset = 0;
            pcb->fdtable[fd].flags = flags;
            return fd;
        }
    }
    return -1;
}
```

open 연산에서는
디스크 접근 없이
디렉토리 해시 캐시를 통해
파일 이름을 inode 번호로 즉시 변환합니다.
이후 PCB의 file descriptor table에
inode 번호와 offset 정보를 저장함으로써,
파일 접근의 기준 상태를 설정했습니다.

### 4.1.2 read 연산 흐름

read 연산은
open 시 설정된 inode와 offset을 기준으로,
논리적 블록 번호를 계산하고
이를 물리적 데이터 블록으로 매핑하는 과정입니다.
이때 inode의 블록 포인터와
버퍼 캐시가 함께 사용됩니다.

**read 연산의 핵심 흐름**

```c
uint32_t lbn = cur / BLOCK_SIZE;
uint32_t inb = cur % BLOCK_SIZE;

uint32_t phys = 0;
inode_get_phys(fs, ino, lbn, &phys);

Buf* b = bcache_getblk(fs, phys);
memcpy(out + done, b->data + inb, chunk);
bcache_brelse(&fs->bcache, b);
```

이 과정에서
inode는 논리 블록 번호를
물리 블록 번호로 변환하는 기준 정보를 제공하며,
버퍼 캐시는 해당 블록이
메모리에 존재하는지 여부를 관리합니다.
이를 통해 read 연산은
디스크 I/O를 최소화하면서
offset 기반으로 순차적인 데이터 접근을 수행합니다.

### 4.1.3 write 연산 흐름

write 연산은
필요한 데이터 블록을 확보한 뒤,
버퍼 캐시를 통해 데이터를 기록하고,
inode 메타데이터를 갱신하는 흐름으로 구현했습니다.
본 구현에서는 overwrite-from-beginning semantics를 사용합니다.

**write 연산의 핵심 흐름**

```c
uint32_t need_blocks =
    (nbytes + BLOCK_SIZE - 1u) / BLOCK_SIZE;

inode_ensure_blocks(fs, ino, need_blocks);

Buf* b = bcache_getblk(fs, phys);
memcpy(b->data + inb, in + done, chunk);
bcache_mark_dirty(b);
bcache_brelse(&fs->bcache, b);

ino->size = nbytes;
inode_free_excess(fs, ino, need_blocks);
```

write 연산에서는
block bitmap을 통해 새로운 데이터 블록을 할당하고,
inode의 direct 또는 indirect block 포인터를 갱신합니다.
이후 버퍼 캐시에 dirty 상태로 기록하여,
실제 디스크 반영은 `fs_sync` 또는 unmount 시점에 수행되도록 했습니다.

### 4.1.4 close 연산 흐름

close 연산은
파일 접근 상태를 해제하는 역할을 수행합니다.
본 구현에서는
PCB의 file descriptor table 엔트리를 초기화함으로써,
해당 파일에 대한 접근을 종료합니다.

**close 연산**

```c
int sys_close(PCB* pcb, FS* fs, int fd) {
    memset(&pcb->fdtable[fd], 0,
           sizeof(pcb->fdtable[fd]));
    return 0;
}
```

close 연산 자체는
디스크 I/O를 발생시키지 않으며,
파일 접근 상태 관리에 집중하도록 단순화했습니다.
실제 데이터와 메타데이터의 영속성은
`fs_sync` 또는 `fs_umount` 단계에서 보장됩니다.

### 4.1.5 연산 흐름 요약

종합하면,
mount 과정에서 준비된
superblock, inode table, 디렉토리 캐시, 버퍼 캐시는
각 파일 연산에서 다음과 같은 역할 분담을 가집니다.

* open: 디렉토리 캐시를 통해 inode 결정
* read: inode + buffer cache를 통한 데이터 접근
* write: bitmap + inode + buffer cache를 통한 데이터 갱신
* close: PCB 기반 접근 상태 해제

이를 통해 파일 시스템의 모든 연산은
mount 시점에 구축된 구조를 그대로 따라 동작하며,
파일 이름에서 데이터 블록까지의 접근 경로가
일관되게 유지되도록 구현했습니다.

## 4.2 File Operation 흐름에 대한 구조적 문제 분석

파일 시스템에서 journaling이나
write-ahead logging(WAL)이 필요한 이유는,
개별 연산이 올바르게 구현되었음에도 불구하고
여러 단계로 분리된 변경 사항이
하나의 완결된 연산으로 보장되지 않기 때문입니다.

즉, 문제의 핵심은
특정 함수의 구현 오류가 아니라,
**여러 처리 단계로 나뉜 변경이
중간 상태를 남길 수 있는 구조**에 있습니다.
본 절에서는 open, read, write, close 연산을
문제 발생 관점에서 분석하여,
왜 추가적인 보장 메커니즘이 필요한지를
구현 구조를 기준으로 설명합니다.

### 4.2.1 문제 요약: 중간 상태가 외부에 노출될 수 있는 구조

현재 구현에서 파일 시스템 연산은
다음과 같은 특성을 가집니다.

* 각 처리 단계는 개별적으로 올바르게 동작합니다.
* 그러나 여러 단계가 결합된 연산은
  하나의 단위로 보장되지 않습니다.
* 그 결과, 연산 도중의 상태가
  디스크에 반영되거나 외부에 관측될 수 있습니다.

journaling과 WAL은
이러한 중간 상태를 허용하지 않기 위해,
여러 변경을 하나의 연산으로 묶어
적용 순서와 범위를 강제하는 기법입니다.

### 4.2.2 open 경로에서 발생할 수 있는 비용 집중 문제

파일 시스템 연산에서
처리 단계의 경계가 명확하지 않을 경우,
디렉토리 탐색과 같은 고비용 작업이
open 경로에 반복적으로 포함됩니다.
이는 파일 시스템 성능 병목의
대표적인 원인입니다.

#### 구현에서의 선택

본 구현에서는 이러한 비용 문제를 줄이기 위해,
디렉토리 탐색과 이름 해석을
open 연산에서 제거하고
mount 시점에 선행 처리하도록 설계하였습니다.
즉, 파일 이름과 inode의 대응 관계를
사전에 결정하여,
open 시점에는 이미 해석된 정보를 조회하도록 합니다.

**mount 시 디렉토리 해시 캐시 구축**

```c
if (parse_root_and_build_cache(fs) != 0) return -1;
```

**open 시 이름 해석**

```c
if (dirhash_get(&fs->dircache, name, len, &ino) != 0) return -1;
```

이를 통해 open 연산의 비용은
디렉토리 크기와 무관해지며,
입력 경로 길이에 따른 성능 변동을 제거할 수 있습니다.

#### 구조적 한계

그러나 이 방식은
디렉토리 구조가 변경되지 않는다는
강한 가정을 전제로 합니다.
rename이나 unlink와 같은
이름 변경을 허용할 경우,
사전에 구축한 정보와 실제 디스크 상태 간에
불일치가 발생합니다.

즉, 본 구현은
성능 문제를 해결하는 대신,
동적 이름 변경이라는 기능을
설계 단계에서 제외한 구조입니다.

### 4.2.3 read/write 경로에서의 처리 분리

read와 write 연산은
논리 블록 번호를 계산한 뒤,
이를 물리 블록으로 변환하고
버퍼 캐시를 통해 실제 데이터를 접근합니다.

**논리 블록에서 물리 블록 접근**

```c
inode_get_phys(fs, ino, lbn, &phys);
Buf* b = bcache_getblk(fs, phys);
```

이 구조를 통해
캐시 히트와 캐시 미스 경로가 분리되며,
디스크 접근 비용은
버퍼 캐시 계층에서 제어됩니다.
그러나 이러한 분리는
성능 최적화를 위한 것이며,
연산 전체의 완료 여부를 보장하지는 않습니다.

### 4.2.4 write 연산에서 발생하는 부분 반영 문제

write 연산에서는
데이터 기록과 inode 메타데이터 갱신이
서로 다른 단계에서 수행됩니다.

**데이터 기록 단계**

```c
memcpy(b->data + inb, in + done, chunk);
bcache_mark_dirty(b);
```

이후 inode의 크기 정보가 갱신됩니다.

**inode 메타데이터 갱신**

```c
ino->size = nbytes;
```

이 두 단계는
논리적으로는 하나의 write 연산에 해당하지만,
디스크 반영 관점에서는
서로 독립적인 변경입니다.
시스템이 이 사이에서 종료될 경우,
데이터와 메타데이터 간의
일관성이 깨질 수 있습니다.

이는 구현 실수가 아니라,
여러 변경을 하나의 연산으로
보장하지 않은 구조적 한계에서 비롯됩니다.

### 4.2.5 왜 journaling과 WAL이 필요한가

journaling과 write-ahead logging은
이러한 구조적 문제를 해결하기 위해,
여러 변경을 사전에 기록하고
완결 여부를 기준으로 반영 여부를 결정합니다.
이를 통해 다음을 보장합니다.

* 변경 사항은 전부 반영되거나,
  전혀 반영되지 않습니다.
* 연산 도중의 상태는
  디스크에 남지 않습니다.
* 비정상 종료 이후에도
  일관된 상태로 복구할 수 있습니다.

이는 성능 향상을 위한 선택이 아니라,
파일 시스템 동작을
예측 가능하고 설명 가능하게 만들기 위한
필수적인 문제 해결 방식입니다.

### 4.2.6 요약

본 구현은
메모리 상에서의 파일 시스템 불변식을
정확히 유지합니다.
그러나 여러 단계로 분리된 변경을
하나의 연산으로 보장하지 않기 때문에,
crash나 확장된 실행 환경에서는
일관성이 깨질 가능성이 존재합니다.

journaling과 write-ahead logging은
이러한 문제를 보완하기 위한
부가 기능이 아니라,
파일 시스템을 신뢰할 수 있는 방식으로
동작시키기 위해 반드시 필요한
구조적 해결책입니다.

## 4.3 Flash 연산 제약이 파일 시스템 구조에 반영되는 방식

앞선 논의에서 확인했듯이,
Linux 기반 파일 시스템 모델은
블록 디바이스 환경에서는
합리적이고 일관된 설계입니다.
그러나 raw NAND Flash가
주 저장 장치로 사용되는 환경에서는,
저장 매체의 물리적 제약이
파일 시스템의 상태 전이 모델 자체를
강하게 제한합니다.

본 절에서는
Flash memory의 기본 연산 제약이
어떻게 littlefs의 내부 구조로
직접 반영되는지를
구체적으로 설명합니다.

### 4.3.1 Flash memory의 기본 연산 모델

Flash memory는
다음과 같은 연산 특성을 가집니다.

* **read** 연산은
  저장된 전하 상태를 판독하며,
  데이터는 항상 읽을 수 있으나
  NAND의 경우 오류를 포함할 수 있습니다.
* **program** 연산은
  비트 값을 **1에서 0으로만**
  변경할 수 있습니다.
* **erase** 연산은
  **0에서 1로의 복원을 담당**하며,
  반드시 **블록 단위로만**
  수행됩니다.

이러한 비대칭성으로 인해
Flash에서는
기존 데이터를 부분적으로 수정하는
overwrite 연산이
물리적으로 성립하지 않습니다.
즉,
상태의 변경은
기존 상태를 갱신하는 과정이 아니라,
**새로운 상태를 생성하는 방식**으로만
안전하게 표현될 수 있습니다.

### 4.3.2 중간 상태가 허용되지 않는 이유

블록 디바이스 기반 파일 시스템에서는,
여러 단계의 갱신이
순차적으로 디스크에 반영되더라도,
overwrite를 통해
이전 상태를 복구할 수 있습니다.
따라서 journaling이나 WAL을 통해
중간 상태를 사후적으로 정리하는
구조가 성립합니다.

그러나 Flash memory에서는
부분적으로 기록된 상태를
되돌리기 위해
블록 단위 erase와
재기록이 필요하며,
이는 높은 지연 시간과
저장 매체 수명 감소로 이어집니다.
결과적으로,
중간 상태를 허용하는 설계 자체가
현실적인 선택이 아닙니다.

이로 인해 littlefs는
중간 상태를 복구하는 구조를
도입하지 않고,
**중간 상태가 생성되지 않도록**
파일 시스템 연산을 제한합니다.

### 4.3.3 Flash 제약이 반영된 littlefs 구조

이러한 연산 제약은
littlefs의 핵심 자료구조에
직접적으로 반영됩니다.
다음은 littlefs의
파일 시스템 상태를 표현하는 구조체의 일부입니다.

**littlefs 파일 시스템 상태 구조체**

```c
typedef struct lfs {
    lfs_cache_t rcache;
    lfs_cache_t pcache;

    lfs_block_t root[2];
    struct lfs_mlist *mlist;

    lfs_gstate_t gstate;
    lfs_gstate_t gdisk;
    lfs_gstate_t gdelta;

    struct lfs_lookahead lookahead;

    const struct lfs_config *cfg;
} lfs_t;
```

이 구조체에는
고정된 inode 테이블이나
전역 비트맵과 같은
중앙 집중적 상태가 존재하지 않습니다.
대신,
파일 시스템의 상태는
로그 형태의 메타데이터와
그 재구성을 통해
동적으로 결정됩니다.

특히,
`root[2]`와
`lfs_gstate`는
Copy-on-Write 기반의
상태 전이를 지원하기 위한 요소로,
새로운 상태가 완전히 기록되기 전까지
기존 상태가 유지되도록 설계되어 있습니다.
이는 Flash memory의
erase-before-write 제약을
직접적으로 반영한 결과입니다.

### 4.3.4 구조적 차이의 필연성

결과적으로,
littlefs가 채택한
로그 기반 상태 전이 모델은
성능 최적화나 구현 취향의 문제가 아니라,
Flash memory의
연산 제약을
정면으로 수용한 결과입니다.

반면,
본 구현과 같은
Linux 기반 파일 시스템 모델은,
overwrite가 가능한
블록 디바이스 환경에서는
합리적으로 동작하지만,
raw NAND Flash 환경에서는
동일한 전제를 유지할 수 없습니다.

따라서 두 파일 시스템 간의 차이는
우수성의 문제가 아니라,
**서로 다른 저장 매체 전제를
기반으로 한 구조적 귀결**로
이해되어야 합니다.

---

# 5. 결론

본 프로젝트에서는 디스크 이미지 기반의
단순 파일 시스템(SimpleFS)을 설계하고 구현함으로써,
파일 시스템의 핵심 구성 요소와
파일 연산 흐름을 코드 수준에서 분석했습니다.

## 파일 시스템 구조 구현 요약

Superblock, inode, directory entry를 중심으로
파일 이름에서 데이터 블록까지 이어지는
기본적인 파일 시스템 구조를 직접 구현했습니다.
mount 시점에 전체 상태를 초기화하고,
이후의 모든 파일 연산이
**동일한 기준 상태(FS 구조체)**를 공유하도록 설계했습니다.
이를 통해 파일 시스템의
**전역 불변식(invariant)**이
어떻게 유지되는지를 명확히 확인했습니다.

## 실행 결과 검증

실행 결과를 통해,
루트 파일 시스템의 마운트,
디렉터리 파싱,
파일 읽기 연산이
의도한 설계에 따라
정상적으로 수행됨을 확인했습니다.
특히 디렉터리 해시 캐시와 버퍼 캐시를 도입함으로써,
open 및 read 경로에서의
**불필요한 디스크 접근을 제거**하고,
파일 접근 성능이
안정적으로 유지됨을 확인했습니다.

## 연산 흐름과 구조적 한계 분석

또한 open, read, write, close 연산을
단순한 함수 호출의 나열이 아닌,
**상태 전이 흐름 관점**에서 분석했습니다.
이를 통해 여러 단계로 분리된 변경이
중간 상태를 외부에 노출할 수 있는
구조적 한계를 확인했습니다.
이 분석을 바탕으로,
journaling이나 write-ahead logging과 같은 기법이
구현상의 선택이 아니라
**구조적으로 요구되는 해결책**임을
이해할 수 있었습니다.

## 저장 매체 제약과 구조적 귀결

아울러 Flash memory 환경을 고려하여,
overwrite가 불가능한 저장 매체의 제약이
파일 시스템 구조에
어떻게 직접적으로 반영되는지를 분석했습니다.
이를 통해 littlefs와 같은 로그 기반 파일 시스템이
성능이나 구현 취향의 문제가 아니라,
**저장 매체의 물리적 특성에 따른
필연적인 설계 귀결**임을 확인했습니다.

## 프로젝트의 범위와 의의

본 프로젝트는
완전한 범용 파일 시스템 구현을 목표로 하지 않고,
파일 시스템의 핵심 개념과
구조적 의사결정을
명확히 드러내는 데 초점을 두었습니다.
그 결과,
자료구조 선택,
연산 흐름 분리,
상태 관리 방식이
어떠한 trade-off를 가지는지
구체적으로 이해할 수 있었습니다.

## 향후 확장 가능성

향후에는 본 구조를 기반으로,
journaling 도입,
동적 디렉터리 갱신,
자원 할당 정책 개선과 같은 확장을 통해,
보다 현실적인 파일 시스템 설계로
발전시킬 수 있을 것입니다.

---

# 부록 A. Execution Log Excerpt

본 부록에서는 실행 결과의 신뢰성을 뒷받침하기 위해,
SimpleFS 실행 시 출력된 로그 일부를 제시합니다.

```text
[fs_mount] root inode:
  ino=2
  mode=0x20777
  size=3264
  indirect_block=-1
  direct_blocks={0,1,2,3,0,0}

== Root directory listing (/) ==
(inode=2, size=3264 bytes)
inode=3  name=file_1
...
inode=94 name=file_92

[pid=1] open(/file_82)
  -> read 41 bytes

Buffer cache stats:
hits=373 misses=13 writebacks=0 evictions=13
```
