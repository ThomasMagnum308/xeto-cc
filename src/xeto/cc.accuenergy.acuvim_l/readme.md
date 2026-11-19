# cc.accuenergy.acuvim_l

Device models for Accuenergy's Acuvim-L Series multifunction power and energy
meters.

Reference: Acuvim-L Multifunction Power and Energy Meter User Manual v4.

## Supported Models

The Acuvim-L Series covers two models that share the same Modbus register
map and BACnet object list:

- **Acuvim-EL**: 0.2% accuracy class (ANSI C12.20 / IEC 62053-22 class 0.5S),
  harmonic measurement to the 63rd order, adds time-of-use (TOU) tariff
  support
- **Acuvim-CL**: 0.5% accuracy class, harmonic measurement to the 31st order

The series-level `AcuvimL*` specs are abstract; model-specific concrete
specs inherit from them. Currently only Acuvim-EL concrete specs are
defined (`AcuvimEL*`); Acuvim-CL specs are planned future work.

## Wiring Configurations

The meter supports five voltage wiring modes (Modicon register 404100): 3LN, 1LN,
2LL, 3LL, and 1LL. The 3LN-2.5 element configuration uses the 3LN setting.
The same Modbus registers are reused across wiring modes; in 2LL/3LL modes
the voltage THD registers report line-to-line THD (V12, V31, V23) rather
than phase THD, so the voltage THD points are modeled on wiring-specific
abstract types:

- `AcuvimLElecAc1LNOr1LLOr3LNMeter`: phase voltage THD
- `AcuvimLElecAc2LLOr3LLMeter`: line-to-line voltage THD

Abstract series-level wiring types: `AcuvimLElecAc3LNMeter`,
`AcuvimLElecAc1LNMeter`, `AcuvimLElecAc1LLMeter`, `AcuvimLElecAc3LLMeter`,
`AcuvimLElecAc2LLMeter`.

Concrete Acuvim-EL types: `AcuvimELElecAc3LNMeter`, `AcuvimELElecAc1LNMeter`,
`AcuvimELElecAc1LLMeter`, `AcuvimELElecAc3LLMeter`, `AcuvimELElecAc2LLMeter`.

## Configuration Assumptions

Point scale factors assume the following meter settings:

- Real-Time Reading mode (Modicon register 404126) set to Primary; values
  include PT/CT ratios with no external multiplier
- Energy Display mode (Modicon register 404122) set to Secondary or Primary
  0.001 mode; energy registers convert with Rx/1000 (`*0.001` scale)
- Power factor convention IEC (Modicon register 404118, factory default)
- Energy calculating mode Full, so power factor points are true power factor

## Coverage

The specs model every register in the Modbus register map of the
Acuvim-L user manual v4.1.5 (August 2024, chapter 6, Tables 6-15 to 6-81),
following `cc.accuenergy.acuvim2`'s conventions. Accuenergy's current manual,
v4.1.13 (November 2026), no longer contains the register map.

- Points with a ph.elec spec use it. Points without one (configuration,
  status, logs, TOU, alarms, I/O modules, SunSpec, max/min records,
  quadrant energies, import plus export totals) are `NumberPoint`,
  `EnumPoint` or `BoolPoint` with `<notHaystack>`, as in acuvim2. A second
  register for a quantity that already has a Haystack point, such as the
  measured neutral current, is typed but `<notHaystack>`.
- Registers that are R/W also carry a `modbusWriteAddr`.
- Apart from real-time metering and energy, the Acuvim-L's register map is
  the Acuvim II's at the same register numbers. The Acuvim-L reads real-time
  values and energy from its own Tables 6-19 (Modicon 408577-) and 6-20
  (408321-), not the Acuvim II's 416385- block, and has no waveform capture.
- Net energies (active, reactive, apparent) are `s4`: the manual's range is
  -999999999 to 999999999 although its table says uint32.

## Raw registers

Some registers have no fixed scale, so they are read raw, as in acuvim2,
typed `NumberPoint` with no unit, and each point's doc gives the manual's
formula:

- Max and min records and TOU current max demand (Tables 6-25, 6-26, 6-34,
  6-35): V = Rx x (PT1/PT2) / 10, A = Rx x (CT1/CT2) / 1000; power and demand
  are whole kW, kvar and kVA. CT2 counts as 1 for 333mV, RCT and mA-output
  CTs (the manual's pulse constant note), so on an RCT meter with CT1 1000
  the raw current is amps.
- Sequence components (Table 6-37): the same voltage and current formulas.
  The table's "Rx/100 %" is copied from the THD table.
- AO setpoint parameter values (Table 6-46): `s2`; the scale follows the AO
  parameter (the note after the table).

## Checked on a meter

An Acuvim-L (RCT, CT1 1000, PT1 and PT2 220.0, Primary parameter mode,
Primary 0.001 energy mode), read with Haxall 4.0.7 and compared with the
meter's web page, November 2026:

- TOU energy (Tables 6-21 to 6-24, 6-61, 6-62) is `*0.001`, as Table 6-20.
- Max and min records follow the formulas above; their timestamps are
  year, month, day, hour, minute, second.
- The 100ms block (Table 6-18) is in Hz, V, A, kW, kvar and kVA.
- Sequence components follow the formulas above (V raw 1230 is 123.0V; A raw
  95 is 0.095A secondary, 95A primary).
- PT1 is `u4` and PT2 `u2`, both in tenths of a volt (2200 is 220.0V), not
  a float.
- Channel 1 communication protocol (0FFEH) reads 0 on Modbus and 2 on
  BACnet MS/TP, as Table 6-15 says; the note after it (1 is BACnet) is
  wrong, and acuvim2 follows the note. A raw register, since the codes
  aren't contiguous.
- AO setpoint 4 reads 10000 (10A, the top of the current range) where the
  manual's default says 1000.

## Unverified

- The TOU error code (780EH) is read only, as in acuvim2; the manual says
  R/W.
- The fourth data log (settings 1500H-15B9H, window 6404H-647EH) is not
  modeled; acuvim2 has three logs.

## Errors in the manual

Where the manual's hex and decimal addresses disagree, the address that
continues the table's sequence is used. Among them: Table 6-29 prints 425DH
for V2(V31) 45th harmonic (4529H); Table 6-46 prints 109H-10E1H for AO2
(10D9H-10E1H); Table 6-18 prints 3403AH for total power factor (303AH);
Table 6-55 gives channel 9 and 10 settings decimal addresses one lower than
their hex; the holiday tables (6-70 to 6-73) have several. Table 6-15 labels
1006H "PT2 (Low 16 bit)"; it is PT1's low word. Table 6-19 swaps the
relationships and units of reactive and apparent power demand, and Table
6-20 gives apparent energy in kVA (kVAh).
