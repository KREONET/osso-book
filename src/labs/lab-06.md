## 실습⑥ 나가는 걸 막는다 — egress 화이트리스팅
🎯 **목표:** 아웃바운드 기본 차단 + squid로 꼭 필요한 곳만.
🖥️ **랩:** 작업 VM. squid 프록시는 랩에 미리 떠있음(allowlist=apt/yum/docker/ghcr).

⌨️ **해보기**
```bash
# 1) 나가는 걸 다 막는다
sudo ufw default deny outgoing
sudo ufw allow out to ⟨10.10.0.9⟩ port 3128    # squid만 예외
sudo ufw allow out 53                           # DNS
# 2) apt/docker가 프록시를 바라보게
echo 'Acquire::http::Proxy "http://⟨10.10.0.9⟩:3128";' | sudo tee /etc/apt/apt.conf.d/01proxy
sudo apt update                                  # ✅ 된다 (allowlist)
docker pull ghcr.io/⟨org/img⟩                    # ✅ 된다
curl -I https://example.com                      # ❌ 막힌다
```
👀 **보이는 것:** apt/docker pull은 성공, 임의 사이트 curl은 타임아웃/거부.
💡 **느낄 것:** 인바운드만 막던 습관이 깨진다. **나가는 걸 화이트리스팅**하면 서버가 뚫려도 공격자가 나갈 구멍이 없다. → Day2 AI에게 이걸 씌운다(복선).

