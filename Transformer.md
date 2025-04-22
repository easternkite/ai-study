## Transformer
> 자연어 처리 및 기타 시퀀스 데이터 처리에 강력한 성능을 보이는 신경망 모델. 이후 BERT, GPT, T5 등 강력한 모델들이 등장하는 기반이 되었다.

* 기존 RNN/LSTM의 한계를 극복하기 위해 등장했으며, 주요 특징은 다음과 같다.
	* Self-Attention (자기 주의 기법)
		* 문장에서 중요한 단어에 집중할 수 있도록 가중치를 주는 방식
		* 예: 나는 사과를 먹었다 -> 사과 와 먹었다 가 밀접한 관계가 있음
		* 기존 RNN 처럼 순차적으로 처리하지 않고, 문장 전체를 한 번에 분석 가능 
* Positional Encoding (위치 인코딩)
	* RNN과 달리 Transformer는 문장의 순서를 직접 기억하지 않음 
	* 대신, Positional Encoding을 추가해 단어의 순서를 기억함
* 병렬 연상 가능
	* RNN/LSTM은 순차적으로 처리하지만, Transformer는 병렬 연산이 가능해 학습 속도가 빠름

### 구조 
> 크게 Encoder와 Decoder로 이루어져 있다.

* Encoder
	* 입력  문장을 Self-Attention과 Feed Forward Neural Network(FFN)를 통해 인코딩 
	* 여러 개의 Encoder Layer로 구성됨
* Decoder
	* Encoder의 출력을 받아서 다음 단어를 예측
	* Self-Attention + Encoder-Decoder Attention + FFN 사용
