# BigGTimer
import React, { useState, useEffect, useRef } from "react";

const TIME_SETS = {
  "PP1": [
    { distance: "7 Yards", time: 15 },
    { distance: "10 Yards", time: 15 },
    { distance: "15 Yards", time: 20 },
    { distance: "25 Yards", time: 35 },
  ],
  "PPC 1500": [
    { distance: "7 Yards (Beidhändig)", time: 20 },
    { distance: "7 Yards (Einhändig)", time: 8 },
    { distance: "15 Yards", time: 90 },
    { distance: "25 Yards", time: 165 },
    { distance: "50 Yards", time: 210 },
  ]
};

export default function ShootingTimerApp() {
  const [discipline, setDiscipline] = useState("PP1");
  const [currentIndex, setCurrentIndex] = useState(0);
  const [timeLeft, setTimeLeft] = useState(0);
  const [isRunning, setIsRunning] = useState(false);
  const [micActive, setMicActive] = useState(false);
  const audioContextRef = useRef(null);
  const analyserRef = useRef(null);
  const sourceRef = useRef(null);
  const dataArrayRef = useRef(null);
  const rafIdRef = useRef(null);

  useEffect(() => {
    if (isRunning && timeLeft > 0) {
      const timer = setTimeout(() => setTimeLeft(timeLeft - 1), 1000);
      return () => clearTimeout(timer);
    }
  }, [isRunning, timeLeft]);

  useEffect(() => {
    if (micActive) startMic();
    else stopMic();
    return () => stopMic();
  }, [micActive]);

  const startMic = async () => {
    audioContextRef.current = new (window.AudioContext || window.webkitAudioContext)();
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    sourceRef.current = audioContextRef.current.createMediaStreamSource(stream);
    analyserRef.current = audioContextRef.current.createAnalyser();
    sourceRef.current.connect(analyserRef.current);
    analyserRef.current.fftSize = 256;
    const bufferLength = analyserRef.current.frequencyBinCount;
    dataArrayRef.current = new Uint8Array(bufferLength);
    detectShot();
  };

  const stopMic = () => {
    if (rafIdRef.current) cancelAnimationFrame(rafIdRef.current);
    if (sourceRef.current) sourceRef.current.disconnect();
    if (audioContextRef.current) audioContextRef.current.close();
  };

  const detectShot = () => {
    analyserRef.current.getByteFrequencyData(dataArrayRef.current);
    const volume = dataArrayRef.current.reduce((a, b) => a + b, 0) / dataArrayRef.current.length;
    if (volume > 50 && !isRunning) startTimer();
    rafIdRef.current = requestAnimationFrame(detectShot);
  };

  const startTimer = () => {
    const currentSet = TIME_SETS[discipline][currentIndex];
    setTimeLeft(currentSet.time);
    setIsRunning(true);
  };

  const stopTimer = () => {
    setIsRunning(false);
  };

  const resetTimer = () => {
    const currentSet = TIME_SETS[discipline][currentIndex];
    setTimeLeft(currentSet.time);
    setIsRunning(false);
  };

  const nextSet = () => {
    const nextIndex = (currentIndex + 1) % TIME_SETS[discipline].length;
    setCurrentIndex(nextIndex);
    setIsRunning(false);
    setTimeLeft(0);
  };

  const previousSet = () => {
    const prevIndex = (currentIndex - 1 + TIME_SETS[discipline].length) % TIME_SETS[discipline].length;
    setCurrentIndex(prevIndex);
    setIsRunning(false);
    setTimeLeft(0);
  };

  const currentSet = TIME_SETS[discipline][currentIndex];
  const isEnding = timeLeft <= currentSet.time * 0.15;

  return (
    <div className="p-6 max-w-xl mx-auto text-center space-y-6 pb-40">
      <h1 className="text-3xl font-extrabold">🎯 Big G's Shooting Timer</h1>

      <select
        className="p-2 border rounded"
        value={discipline}
        onChange={(e) => {
          setDiscipline(e.target.value);
          setCurrentIndex(0);
          setIsRunning(false);
          setTimeLeft(0);
        }}
      >
        <option value="PP1">PP1</option>
        <option value="PPC 1500">PPC 1500</option>
      </select>

      <div className="text-xl font-semibold">
        {currentSet.distance}: {currentSet.time} sec
      </div>

      <div className={`text-6xl font-mono ${isEnding ? "text-red-600" : "text-black"}`}>
        ⏱ {timeLeft}s
      </div>

      <div className="space-x-2">
        <button
          className="bg-green-500 text-white px-4 py-2 rounded"
          onClick={() => setMicActive(!micActive)}
        >
          {micActive ? "🎙️ Mic aus" : "🎙️ Mic an"}
        </button>

        <button
          className="bg-blue-500 text-white px-4 py-2 rounded"
          onClick={previousSet}
        >
          ⬅️ Letzte Serie
        </button>

        <button
          className="bg-blue-500 text-white px-4 py-2 rounded"
          onClick={nextSet}
        >
          ➡️ Nächste Serie
        </button>
      </div>

      <div className="space-x-2 pt-10">
        <button
          className="bg-yellow-500 text-white px-6 py-3 rounded text-lg"
          onClick={startTimer}
        >
          ▶️ Start
        </button>

        <button
          className="bg-red-500 text-white px-6 py-3 rounded text-lg"
          onClick={stopTimer}
        >
          ⏹ Stop
        </button>

        <button
          className="bg-gray-500 text-white px-6 py-3 rounded text-lg"
          onClick={resetTimer}
        >
          🔁 Reset
        </button>
      </div>
    </div>
  );
}
