# SimpleFS: 파일 시스템 구조 설계 및 구현 보고서

---

## 1. 프로젝트 개요

### 1.1 프로젝트 목적

디스크 이미지 기반의 간단한 파일 시스템(SimpleFS)을 설계하고 구현했습니다. 이를 통해 운영체제에서 파일 시스템이 수행하는 메타데이터 관리, 데이터 접근, 상태 전이 과정을 코드 수준에서 직접 확인하는 것을 목표로 했습니다.

파일 시스템은 Superblock, inode, directory entry와 같은 핵심 자료구조를 중심으로 설계했으며, 파일 이름이 디렉터리를 통해 inode로 연결되고 실제 데이터 블록에 매핑되는 구조를 구현했습니다. 또한 open, read, write, close 연산을 직접 구현하여 파일 시스템의 기본 동작을 검증했습니다.

추가적으로 디렉터리 해시 캐시와 buffer cache를 도입하여 파일 접근 성능을 개선했으며, 데이터 블록 할당자와 디스크 이미지 생성 도구를 분리 구현하여 구조적인 완성도를 높였습니다. 이 과정을 통해 파일 시스템의 내부 동작과 설계 요소를 보다 명확히 이해할 수 있었습니다.

---

## 2. Execution Results and Validation

본 절에서는 SimpleFS 구현의 실행 결과를 제시하고, 파일 시스템의 마운트, 디렉터리 파싱, 파일 읽기 동작이 의도한 설계에 따라 정상적으로 수행됨을 검증합니다.

---

### 2.1 Filesystem Mount State

`fs_mount()` 실행 결과,

파일 시스템 메타데이터가 정상적으로 파싱되었음을 확인합니다.

* 루트 inode 번호는 **inode 2**로 설정되어 있으며, superblock의 `first_inode` 필드와 일치합니다.
* 루트 디렉터리의 크기는 **3264 bytes**로, 디렉터리 엔트리들의 총 크기와 합리적으로 일치합니다.
* 루트 디렉터리는 간접 블록을 사용하지 않으며, 4개의 직접 데이터 블록만을 사용합니다.

이를 통해 superblock, inode 테이블, 데이터 블록 매핑 정보가 일관된 상태로 로드되었음을 확인합니다.

---

### 2.2 Root Directory Listing

마운트 직후 루트 디렉터리 목록을 출력한 결과는 다음과 같습니다.

* `.` 및 `..` 엔트리가 정상적으로 존재합니다.
* `file_1`부터 `file_92`까지의 파일이 중복 없이 한 번씩만 출력됩니다.
* inode 번호는 연속적으로 증가하며, 디렉터리 파싱 과정에서 무한 반복이나 잘못된 엔트리 해석은 발생하지 않습니다.

이를 통해 루트 디렉터리 파싱 로직과 디렉터리 해시 캐시가 정상적으로 동작함을 확인합니다.

---

### 2.3 File Read Operations

무작위로 선택된 여러 파일에 대해 `open()` 및 `read()`를 수행한 결과는 다음과 같습니다.

* 각 파일은 정상적으로 열립니다.
* 요청한 바이트 수만큼의 데이터가 반환됩니다.
* 동일 파일을 반복하여 읽을 경우 항상 동일한 내용이 반환됩니다.

이는 inode에서 데이터 블록으로의 매핑이 올바르며, 읽기 경로에서 메모리 손상이나 잘못된 블록 접근이 발생하지 않음을 의미합니다.

---

### 2.4 Buffer Cache Statistics

실행 종료 시 출력된 버퍼 캐시 통계를 통해 다음과 같은 특성을 확인합니다.

* 초기 접근 시 일부 cache miss가 발생합니다.
* 이후 반복 접근에서는 높은 cache hit 비율을 보입니다.
* 쓰기 연산이 없으므로 writeback 횟수는 0입니다.
* eviction 횟수는 LRU 정책에 따라 합리적인 수준을 유지합니다.

이를 통해 버퍼 캐시가 정상적으로 초기화되고, 의도한 정책에 따라 동작함을 확인합니다.

---

### 2.5 Discussion

본 실행 결과는 SimpleFS 구현이 파일 시스템의 핵심 구조적 불변식(invariant)을 정상적으로 만족하고 있음을 보여줍니다.

1. 루트 inode가 superblock 메타데이터와 일관되게 해석되며, 모든 디렉터리 및 파일 접근이 해당 inode를 기준으로 수행됩니다.
2. 루트 디렉터리 파싱 과정에서 중복 출력이나 무한 반복이 발생하지 않으며, 각 디렉터리 엔트리가 정확히 한 번씩만 처리됩니다.
3. 파일 읽기 결과가 파일별로 일관되게 유지되며, 버퍼 캐시를 포함한 읽기 경로 전반에서 데이터 무결성이 보장됩니다.

이를 통해 본 구현이 읽기 전용 파일 시스템으로서 안정적인 상태 전이를 수행함을 확인합니다.

---

### 2.6 Summary

위 실행 결과를 통해 SimpleFS의 마운트 과정, 루트 디렉터리 파싱, 파일 읽기 경로가 정상적으로 구현되었음을 확인합니다.

현재 구현은 읽기 전용 파일 시스템으로서 일관된 상태와 안정적인 동작을 보장합니다.

---

## 3. 파일 시스템 구조 및 설계 선택

SimpleFS의 전체 구조와 각 구성 요소의 설계 의도입니다.
비트맵 기반 자원 관리와 캐시 구조를 중심으로 파일 시스템의 설계 방향을 정리했습니다.

---

### 3.1 디스크 레이아웃 및 전체 구조

```c
typedef struct {
    FILE* disk;
    struct super_block sb;
    struct inode inode_table[MAX_INODES];

    uint8_t* inode_bm;
    uint8_t* block_bm;

    uint32_t data_blocks;
    uint32_t inode_count;

    DirHash  dircache;
    BufCache bcache;
} FS;
```

* Block 0: superblock
* Block 1~7: inode table
* Block 8 이후: data block

파일 시스템 전체 상태는 `FS` 구조체에서 관리됩니다.

---

### 3.2 Superblock 및 비트맵 기반 자원 관리

#### Superblock

```c
struct super_block {
    uint32_t block_size;
    uint32_t num_inodes;
    uint32_t num_blocks;
    uint32_t first_data_block;
    uint8_t padding[960];
};
```

* 파일 시스템 전역 상태 저장
* bitmap 포함

---

#### Bitmap 기반 관리

* inode / block 상태를 bit로 관리
* O(1) 접근

단점:

* 선형 탐색 필요
* free 감소 시 성능 저하
* superblock 손상 시 전체 영향

---

### 3.3 Inode 구조

```c
struct inode {
    uint32_t size;
    int32_t indirect_block;
    uint16_t blocks[6];
};
```

* small file → direct block
* large file → indirect block

---

## 4. 파일 연산 흐름

### 4.1 Open

```c
dirhash_get(...)
```

* 디렉터리 캐시 기반 탐색
* 디스크 접근 없음

---

### 4.2 Read

```c
inode_get_phys(...)
bcache_getblk(...)
```

* inode → block 매핑
* buffer cache 사용

---

### 4.3 Write

```c
alloc_dblk(...)
bcache_mark_dirty(...)
```

* block bitmap 사용
* inode 갱신

---

### 4.4 Close

* FD 해제
* I/O 없음

---

## 5. 구조적 문제 분석

### 5.1 Partial Write 문제

* data write
* inode update

→ 분리 수행

→ crash 시 inconsistency

---

### 5.2 해결

* journaling
* WAL

---

## 6. Flash 환경 차이

Flash:

* overwrite 불가
* erase 필요

→ copy-on-write 구조 필요

---

## 7. 결론

* 파일 시스템 구조 이해
* 상태 전이 분석
* 캐시 구조 검증
* 구조적 한계 확인

---

## Appendix

```text
root inode: 2
size: 3264
hits=373 misses=13
```
