# DuckDB-GeoParquet-Source-Cooperative

Matt Forrest의 Query global buildings using DuckDB, GeoParquet, and Source Cooperative 유튜브 콘텐츠를 바탕으로, Source Cooperative의 거대 공간 데이터셋을 DuckDB와 Python을 이용해 효율적으로 분석하고 시각화하는 방법을 다룹니다.
https://www.youtube.com/watch?v=gDvDo0oNtmw&t=341s

🚀 Key Features
Zero-Infrastructure: 로컬 서버나 복잡한 DB 설치 없이 클라우드 데이터에 직접 쿼리합니다.

High Performance: GeoParquet 포맷과 DuckDB의 컬럼 지향 엔진을 결합해 수억 개의 건물 데이터를 초 단위로 처리합니다.

3D & 2D Visualization: Pydeck을 이용한 3D Extrusion 가시화 및 Folium 기반의 위성 지도 시각화를 지원합니다.

🛠 Tech Stack
Platform: Source Cooperative (Cloud Native Data Sharing)

Database: DuckDB (with Spatial & HTTPFS extensions)

Data Format: GeoParquet

Visualization: Pydeck (3D), Folium (2D Satellite)

📖 Deep Dive
1. Source Cooperative: 지리 데이터의 유튜브
CloudNativePG와 Radiant Earth가 만든 플랫폼으로, 데이터를 다운로드하지 않고도 필요한 부분만 골라 분석할 수 있는 환경을 제공합니다.

vida vs cholmes 데이터셋:

vida: Google과 Microsoft 데이터를 합쳐 S2 Cell(격자) 단위로 파티셔닝되어 있어 좌표 기반 검색에 유리합니다.

cholmes: Google 원본을 국가별(country_iso)로 분류하여 국가 단위 분석에 최적화되어 있습니다.

2. 왜 DuckDB + GeoParquet 인가?
Parquet은 열(Column) 기반 저장 방식입니다. DuckDB는 쿼리에 필요한 컬럼만 읽어오는 Predicate Pushdown 기능을 통해 클라우드 전송량을 획기적으로 줄입니다.

💻 Code Snippets
✅ 클라우드 데이터 직접 쿼리 (Cloud-Native)
parquet_scan을 통해 S3에서 필요한 국가 데이터만 즉시 테이블로 생성합니다.

Python

import duckdb

con = duckdb.connect()
con.execute("INSTALL spatial; LOAD spatial;")
con.execute("INSTALL httpfs; LOAD httpfs;")

# 대한민국(KOR) 건물 데이터 1만 개 스캔
con.execute("""
    CREATE OR REPLACE TABLE kr_buildings AS 
    SELECT * FROM parquet_scan('s3://.../country_iso=KOR/*.parquet') 
    LIMIT 10000;
""")
✅ 안정적인 GeoPackage 저장 (Error Handling)
UNKNOWN Z 타입 에러와 STRUCT 타입 호환성 문제를 해결하는 방식입니다.

Python

# bbox(STRUCT)는 제외하고, TRY_CAST로 불량 지오메트리 처리
con.execute("""
    COPY (
        SELECT * EXCLUDE (geometry, bbox), 
        TRY_CAST(geometry AS GEOMETRY) AS geom 
        FROM kr_buildings 
        WHERE geom IS NOT NULL
    ) TO 'korea_buildings.gpkg' WITH (FORMAT GDAL, DRIVER 'GPKG');
""")
✅ 2D 위성 지도 시각화 (Folium)
Google Satellite 타일을 배경으로 건물 폴리곤을 시각화합니다.

Python

import folium

m = folium.Map(location=[37.5665, 126.9780], zoom_start=18)
folium.TileLayer(
    tiles='https://mt1.google.com/vt/lyrs=s&x={x}&y={y}&z={z}',
    attr='Google'
).add_to(m)
