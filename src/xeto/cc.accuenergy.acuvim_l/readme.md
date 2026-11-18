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

## Unverified

- TOU energy registers (Tables 6-21 to 6-24 and 6-61, 6-62): the manual gives
  no resolution. They are scaled like the energy registers of Table 6-20
  (`*0.001`, Primary 0.001 mode). On a meter the TOU tariffs should sum to
  the matching Table 6-20 total.
- Max and min records (Tables 6-34, 6-35) are `int` registers scaled by the
  meter's PT and CT ratios, (Rx*(PT1/PT2))/10 and so on, which no fixed
  `scale` can express. They are read raw, as in acuvim2.

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
