# WebGL2 & Shader 기본 정리

이 문서는 WebGL2와 three.js를 비교하고, 기본 파이프라인 및 코드 구조를 이해하기 위해 공부한 내용을 정리한 것입니다.  

---

## 1. WebGL2와 three.js 차이

- **WebGL2**: 브라우저에서 GPU를 직접 제어할 수 있는 저수준 API (OpenGL ES 3.0 기반).
  - 정점 버퍼, 셰이더, 행렬 계산 등을 직접 관리해야 한다.
  - 큐브 하나만 그려도 수십~수백 줄 코드가 필요하다.

- **three.js**: WebGL2 위에 구축된 고수준 라이브러리/엔진.
  - `Scene`, `Camera`, `Renderer`, `Mesh` 같은 객체 제공.
  - 기본 도형, 재질, 조명, 모델 로더 등을 쉽게 사용할 수 있다.
  - 코드가 훨씬 간결하고 생산성이 높음.

비유:  
- WebGL2 = 벽돌, 철근으로 직접 집 짓기  
- three.js = 조립식 블록으로 집 짓기  

---

## 2. GPU 파이프라인 흐름

1. **정점 셰이더(Vertex Shader)**  
   - 각 정점의 위치, 속성을 계산 (행렬 변환 등).

2. **프리미티브 조립 & 래스터화**  
   - 정점들을 삼각형으로 묶고, 화면 픽셀로 쪼갬.

3. **프래그먼트 셰이더(Fragment Shader)**  
   - 각 픽셀(프래그먼트)의 최종 색 계산.

4. **테스트 & 출력**  
   - 깊이 테스트, 블렌딩 등을 거쳐 프레임버퍼에 기록.

---

## 3. 셰이더 간 데이터 전달 (in/out, varyings)

- 정점 셰이더에서 `out vColor = aColor;`  
- 프래그먼트 셰이더에서 `in vColor;`  
- 삼각형 내부 픽셀은 GPU가 자동으로 **보간(interpolation)** 해줌.

즉, 정점 단위로 준 색이 픽셀 단위에서 자연스럽게 이어진다.

---

## 4. uniform과 getUniformLocation

- `uniform` = 드로우 호출 동안 모든 셰이더에서 공통으로 쓰이는 값 (행렬, 조명, 시간 등).
- JS에서 `gl.getUniformLocation(program, "uProjection")`으로 **위치 핸들**을 얻는다.
- 이후 `gl.uniformMatrix4fv(location, false, matrix)` 같은 함수로 값을 업로드.

---

## 5. 데이터 구조 (positions, colors, indices)

- **positions**: 큐브 8개 꼭짓점 좌표
  ```
  0: (-1, -1, -1)  back-left-bottom
  1: ( 1, -1, -1)  back-right-bottom
  2: ( 1,  1, -1)  back-right-top
  3: (-1,  1, -1)  back-left-top
  4: (-1, -1,  1)  front-left-bottom
  5: ( 1, -1,  1)  front-right-bottom
  6: ( 1,  1,  1)  front-right-top
  7: (-1,  1,  1)  front-left-top
  ```

- **indices**: 정점 번호로 삼각형을 정의
  ```
  0,1,2,  2,3,0,   // back face
  4,5,6,  6,7,4,   // front face
  0,4,7,  7,3,0,   // left face
  1,5,6,  6,2,1,   // right face
  3,2,6,  6,7,3,   // top face
  0,1,5,  5,4,0    // bottom face
  ```

- **colors**: 각 정점에 색을 지정 → 삼각형 내부 픽셀은 자동으로 보간.

---

## 6. VBO, EBO, VAO

- **VBO (Vertex Buffer Object)**: 정점 데이터 (위치, 색 등)를 GPU 메모리에 올림.
- **EBO (Element Buffer Object)**: 인덱스 배열을 GPU에 저장.
- **VAO (Vertex Array Object)**: VBO와 EBO의 연결 상태(속성 슬롯 ↔ 버퍼 매핑)를 저장.

비유:  
- VBO/EBO = 원자재 (좌표표, 색표, 인덱스표)  
- VAO = “어떻게 읽을지” 설명서 (속성 슬롯과 포맷 정보)

---

## 7. vertexAttribPointer & enableVertexAttribArray

- `gl.vertexAttribPointer(index, size, type, normalized, stride, offset)`  
  → 현재 바인딩된 VBO를 **어떤 형식으로 속성 슬롯에 연결할지** 정의.

- `gl.enableVertexAttribArray(index)`  
  → 해당 속성 슬롯을 “활성화”해서 VBO에서 실제 데이터를 읽도록 함.  
  (안 켜면 그 속성은 상수값(0,0,0,1)로 취급)

---

## 8. 기타 WebGL 함수

- `gl.enable(gl.DEPTH_TEST)`  
  → 깊이 테스트 켜기 (가까운 면이 멀리 있는 면을 가림).

- `gl.clearColor(r,g,b,a)`  
  → 화면을 지울 때 사용할 배경색 지정.

- `gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT)`  
  → 프레임버퍼와 깊이 버퍼를 초기화.

---

## 9. Projection vs ModelView

- **Projection**: 카메라 렌즈. 3D → 2D 투영, 원근감 조절.  
- **ModelView**: 물체 위치/회전/이동(Model) + 카메라 시점(View).  
- 최종 좌표 변환은 `Projection * ModelView * position`.

---

## 10. 애니메이션 루프 순서

```js
function render(t) {
  // 1) 시간 갱신
  t *= 0.001;

  // 2) 캔버스 크기/viewport 갱신
  resizeCanvas();

  // 3) 화면 초기화
  gl.clear(gl.COLOR_BUFFER_BIT | gl.DEPTH_BUFFER_BIT);

  // 4) 행렬 계산 (modelView, projection)
  ...

  // 5) 프로그램/VAO 바인딩
  gl.bindVertexArray(vao);
  gl.useProgram(program);

  // 6) 드로우 호출
  gl.drawElements(gl.TRIANGLES, indices.length, gl.UNSIGNED_SHORT, 0);

  // 7) 다음 프레임 예약
  requestAnimationFrame(render);
}
```

---

## 11. useProgram을 루프에서 다시 호출하는 이유

- `useProgram`은 전역 상태. 다른 코드에서 바뀌었을 수 있으므로, **매 드로우마다 안전하게 지정**하는 습관을 가진다.
- 초기화 단계에서는 uniform 업로드 때문에 한 번 호출.
- 루프 단계에서는 draw 직전에 다시 호출해서 상태를 보장.

---

## 12. 전체 흐름 요약

```
1) 데이터 준비 (positions, colors, indices)
2) VBO/EBO 생성 및 업로드
3) VAO에 속성 연결 (vertexAttribPointer + enable)
4) 셰이더 작성 → 컴파일/링크 → program 생성
5) uniform location 얻고 값 업로드
6) 루프 시작:
   - 캔버스/viewport 갱신
   - 화면 초기화(clear)
   - 행렬 계산 & uniform 업로드
   - useProgram + bindVertexArray
   - drawElements 실행
   - requestAnimationFrame(render)
```

---

이 정리는 개인 학습 목적으로 작성된 내용이며, 팀원들과 WebGL2 기본 개념을 공유하기 위해 README에 올릴 수 있도록 작성되었습니다.
