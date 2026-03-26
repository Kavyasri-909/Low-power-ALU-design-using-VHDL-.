# Low-power-ALU-design-using-VHDL-.
Design and verify a parameterizable, low-power ALU in Verilog (8/16/32-bit) that supports core arithmetic/logic operations and integrates operand-isolation, clock-enable (RTL clock gating), and a “low-power mode”. Provide a reproducible flow for simulation (Icarus/Verilator), power proxy via toggle/VCD/SAIF.
library IEEE;
use IEEE.STD_LOGIC_1164.ALL;
use IEEE.NUMERIC_STD.ALL;

entity alu is
  generic (
    WIDTH       : integer := 32;
    USE_CLA     : integer := 0;
    APPROX_LSB  : integer := 0
  );
  port (
    clk     : in  std_logic;
    rst_n   : in  std_logic;
    en      : in  std_logic;
    lp_mode : in  std_logic;
    A       : in  std_logic_vector(WIDTH-1 downto 0);
    B       : in  std_logic_vector(WIDTH-1 downto 0);
    OPC     : in  std_logic_vector(3 downto 0);
    Y       : out std_logic_vector(WIDTH-1 downto 0);
    Z       : out std_logic;
    N       : out std_logic;
    C       : out std_logic;
    V       : out std_logic
  );
end entity;

architecture rtl of alu is

  -- opcode decoding
  signal do_add, do_sub, do_and, do_or, do_xor, do_nor : std_logic;
  signal do_sll, do_srl, do_sra, do_slt, do_a, do_b    : std_logic;

  signal A_add, B_add, A_log, B_log, A_sh, B_sh : std_logic_vector(WIDTH-1 downto 0);

  signal addB : std_logic_vector(WIDTH-1 downto 0);
  signal cin  : std_logic;

  signal sum  : std_logic_vector(WIDTH-1 downto 0);
  signal cout, v_of : std_logic;

  signal y_and, y_or, y_xor, y_nor : std_logic_vector(WIDTH-1 downto 0);
  signal y_sll, y_srl, y_sra       : std_logic_vector(WIDTH-1 downto 0);
  signal y_slt                     : std_logic_vector(WIDTH-1 downto 0);

  signal y_next : std_logic_vector(WIDTH-1 downto 0);

  signal Z_n, N_n, C_n, V_n : std_logic;

  signal shamt : integer range 0 to WIDTH-1;

begin

  -- opcode decode
  do_add <= '1' when OPC = "0000" else '0';
  do_sub <= '1' when OPC = "0001" else '0';
  do_and <= '1' when OPC = "0010" else '0';
  do_or  <= '1' when OPC = "0011" else '0';
  do_xor <= '1' when OPC = "0100" else '0';
  do_nor <= '1' when OPC = "0101" else '0';
  do_sll <= '1' when OPC = "0110" else '0';
  do_srl <= '1' when OPC = "0111" else '0';
  do_sra <= '1' when OPC = "1000" else '0';
  do_slt <= '1' when OPC = "1001" else '0';
  do_a   <= '1' when OPC = "1010" else '0';
  do_b   <= '1' when OPC = "1011" else '0';

  -- operand isolation
  A_add <= A when (do_add='1' or do_sub='1') else (others => '0');
  B_add <= B when (do_add='1' or do_sub='1') else (others => '0');

  A_log <= A when (do_and='1' or do_or='1' or do_xor='1' or do_nor='1') else (others => '0');
  B_log <= B when (do_and='1' or do_or='1' or do_xor='1' or do_nor='1') else (others => '0');

  A_sh <= A when (do_sll='1' or do_srl='1' or do_sra='1') else (others => '0');
  B_sh <= B when (do_sll='1' or do_srl='1' or do_sra='1') else (others => '0');

  -- add/sub
  addB <= not B_add when do_sub='1' else B_add;
  cin  <= '1' when do_sub='1' else '0';

  -- FIXED ADDER
  process(A_add, addB, cin)
    variable tmp : unsigned(WIDTH downto 0);
    variable cin_ext : unsigned(WIDTH downto 0);
  begin
    if cin = '1' then
      cin_ext := (0 => '1', others => '0');
    else
      cin_ext := (others => '0');
    end if;

    tmp := ('0' & unsigned(A_add)) + ('0' & unsigned(addB)) + cin_ext;

    sum  <= std_logic_vector(tmp(WIDTH-1 downto 0));
    cout <= tmp(WIDTH);

    -- FIXED overflow
  v_of <= (A_add(WIDTH-1) xnor addB(WIDTH-1)) and
        (A_add(WIDTH-1) xor sum(WIDTH-1));
  end process;

  -- logic
  y_and <= A_log and B_log;
  y_or  <= A_log or B_log;
  y_xor <= A_log xor B_log;
  y_nor <= not (A_log or B_log);

  -- shift amount
  shamt <= 1 when lp_mode='1' else to_integer(unsigned(B_sh(4 downto 0)));

  y_sll <= std_logic_vector(shift_left(unsigned(A_sh), shamt));
  y_srl <= std_logic_vector(shift_right(unsigned(A_sh), shamt));
  y_sra <= std_logic_vector(shift_right(signed(A_sh), shamt));

  -- slt
  y_slt <= (others => '0');
  y_slt(0) <= '1' when signed(A) < signed(B) else '0';

  -- mux
  y_next <= sum   when (do_add='1' or do_sub='1') else
            y_and when do_and='1' else
            y_or  when do_or='1' else
            y_xor when do_xor='1' else
            y_nor when do_nor='1' else
            y_sll when do_sll='1' else
            y_srl when do_srl='1' else
            y_sra when do_sra='1' else
            y_slt when do_slt='1' else
            A     when do_a='1' else
            B     when do_b='1' else
            (others => '0');

  -- flags
  Z_n <= '1' when y_next = std_logic_vector(to_unsigned(0, WIDTH)) else '0';
  N_n <= y_next(WIDTH-1);
  C_n <= cout when (do_add='1' or do_sub='1') else '0';
  V_n <= v_of when (do_add='1' or do_sub='1') else '0';

  -- registers
  process(clk, rst_n)
  begin
    if rst_n='0' then
      Y <= (others => '0');
      Z <= '0'; N <= '0'; C <= '0'; V <= '0';
    elsif rising_edge(clk) then
      if en='1' then
        Y <= y_next;
        Z <= Z_n;
        N <= N_n;
        C <= C_n;
        V <= V_n;
      end if;
    end if;
  end process;

end architecture;
