// pages/index.js (Calculador de Vigas Genéricas con múltiples cargas y tipos de apoyo para Vercel)

import { useState } from "react";
import { LineChart, Line, CartesianGrid, XAxis, YAxis, Tooltip, ReferenceDot } from "recharts";

export default function Home() {
  const [longitud, setLongitud] = useState(6);
  const [EI, setEI] = useState(210e9 * 8.5e-6);
  const [cargas, setCargas] = useState([{ valor: 50, posicion: 3 }]);
  const [tipoApoyo, setTipoApoyo] = useState("simple"); // simple o empotrada

  // Reacciones aproximadas para viga simplemente apoyada con múltiples cargas
  const RA = cargas.reduce((acc, carga) => acc + carga.valor * 1000 * (longitud - carga.posicion) / longitud, 0);
  const RB = cargas.reduce((acc, carga) => acc + carga.valor * 1000 * carga.posicion / longitud, 0);

  const puntos = Array.from({ length: 100 }, (_, i) => {
    const x = (i / 99) * longitud;
    let M = 0;
    cargas.forEach(c => {
      if (x <= c.posicion) M += RA * x;
      else M += RA * x - c.valor * 1000 * (x - c.posicion);
    });
    const MEI = M / EI;
    return { x: parseFloat(x.toFixed(2)), M, MEI };
  });

  const area = puntos.reduce((acc, curr, i, arr) => {
    if (i === 0) return 0;
    const dx = arr[i].x - arr[i - 1].x;
    return acc + ((arr[i].MEI + arr[i - 1].MEI) / 2) * dx;
  }, 0);

  const momentoEstatico = puntos.reduce((acc, curr, i, arr) => {
    if (i === 0) return 0;
    const dx = arr[i].x - arr[i - 1].x;
    const avgMEI = (arr[i].MEI + arr[i - 1].MEI) / 2;
    const xc = (arr[i].x + arr[i - 1].x) / 2;
    return acc + avgMEI * dx * xc;
  }, 0);

  const xCentroidal = momentoEstatico / area;
  const theta = area;

  const tangente = puntos.map(p => ({ x: p.x, tangente: theta * (p.x - xCentroidal) }));
  const deflexion = puntos.map(p => {
    let delta = 0;
    cargas.forEach(c => {
      delta += (p.x < c.posicion ? RA * p.x * (longitud - p.x) / (2 * EI) : RA * p.x * (longitud - p.x) / (2 * EI) - c.valor * 1000 * (p.x - c.posicion) ** 2 / (2 * EI));
    });
    return { x: p.x, delta };
  });

  const addCarga = () => setCargas([...cargas, { valor: 0, posicion: 0 }]);
  const updateCarga = (index, field, value) => {
    const nuevasCargas = [...cargas];
    nuevasCargas[index][field] = parseFloat(value);
    setCargas(nuevasCargas);
  };

  return (
    <div className="p-6 grid grid-cols-1 gap-6">
      <div className="p-4 border rounded-xl shadow-md">
        <h2 className="text-xl font-bold mb-4">Calculador de Vigas Genéricas</h2>
        <div className="grid grid-cols-2 gap-4 mb-4">
          <div>
            <label>Longitud (m)</label>
            <input type="number" value={longitud} onChange={e => setLongitud(parseFloat(e.target.value))} className="border p-2 w-full" />
          </div>
          <div>
            <label>Tipo de apoyo</label>
            <select value={tipoApoyo} onChange={e => setTipoApoyo(e.target.value)} className="border p-2 w-full">
              <option value="simple">Simple</option>
              <option value="empotrada">Empotrada</option>
            </select>
          </div>
        </div>
        <h3 className="font-bold mb-2">Cargas</h3>
        {cargas.map((c, i) => (
          <div key={i} className="grid grid-cols-3 gap-2 mb-2">
            <input type="number" value={c.valor} onChange={e => updateCarga(i, 'valor', e.target.value)} placeholder="kN" className="border p-2" />
            <input type="number" value={c.posicion} onChange={e => updateCarga(i, 'posicion', e.target.value)} placeholder="Posición (m)" className="border p-2" />
          </div>
        ))}
        <button onClick={addCarga} className="mt-2 p-2 bg-blue-500 text-white rounded">Agregar Carga</button>
      </div>

      <div className="p-4 border rounded-xl shadow-md">
        <h3 className="text-lg font-bold mb-2">Diagramas y Resultados</h3>
        <LineChart width={750} height={400} data={puntos}>
          <CartesianGrid stroke="#ccc" />
          <XAxis dataKey="x" label={{ value: "x (m)", position: "insideBottomRight" }} />
          <YAxis label={{ value: "Valores", angle: -90, position: "insideLeft" }} />
          <Tooltip />
          <Line type="monotone" dataKey="M" stroke="#8884d8" dot={false} name="M(x)" />
          <Line type="monotone" dataKey="MEI" stroke="#82ca9d" dot={false} name="M/EI" />
          <ReferenceDot x={xCentroidal.toFixed(2)} y={0} r={6} fill="red" label={{ value: "x̄", position: "top" }} />
        </LineChart>

        <LineChart width={750} height={300} data={tangente} className="mt-6">
          <CartesianGrid stroke="#ccc" />
          <XAxis dataKey="x" />
          <YAxis />
          <Tooltip />
          <Line type="monotone" dataKey="tangente" stroke="#ff7300" dot={false} name="Línea Tangente" />
        </LineChart>

        <LineChart width={750} height={300} data={deflexion} className="mt-6">
          <CartesianGrid stroke="#ccc" />
          <XAxis dataKey="x" />
          <YAxis />
          <Tooltip />
          <Line type="monotone" dataKey="delta" stroke="#387908" dot={false} name="Deflexión relativa" />
        </LineChart>
      </div>
    </div>
  );
}
