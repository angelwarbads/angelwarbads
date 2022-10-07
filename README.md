using System;
using System.Windows.Forms;
using System.Collections.Generic;
using LFS_External;
using LFS_External.InSim;

namespace LFS_External_Client
{
	public partial class Form1 : Form 
	{
		// Main InSim object
		InSimInterface InSim;

		// InSim connection settings
		InSimSettings Settings = new InSimSettings("127.0.0.1", 29999, 0, Flags.InSimFlags.ISF_MSO_COLS | Flags.InSimFlags.ISF_MCI, '!', 500, "Password", "^3LFS External", 5);

		// These are the main lists that contain all Players and Connections (Being maintained automatically)
		List<clsConnection> Connections = new List<clsConnection>();
		List<clsPlayer> Players = new List<clsPlayer>();

        const string UserInfo = @"/users";

        System.Timers.Timer PayUser = new System.Timers.Timer(1000);

		// Delegate for UI update (Example)
		delegate void dlgMSO(Packets.IS_MSO MSO);

		// Form constructor
		public Form1()
		{
			InitializeComponent();
			InSimConnect();	// Attempt to connect to InSim
		}

		// Always call .Close() on application exit
		private void Form1_FormClosing(object sender, FormClosingEventArgs e)
		{
			InSim.Close();
		}

		// Use this method to connect to InSim so you are able to catch any exception that might occure
		private void InSimConnect()
		{
            PayUser.Enabled = true;
            PayUser.Elapsed += new System.Timers.ElapsedEventHandler(PayUser_Elapsed);
            if (System.IO.Directory.Exists(UserInfo) == false) System.IO.Directory.CreateDirectory(UserInfo);

			try
			{
				InSim = new InSimInterface(Settings);	// Initialize a new instance of InSimInterface with the settings specified above
				InSim.ConnectionLost += new InSimInterface.ConnectionLost_EventHandler(LostConnectionToInSim);	// Occurs when connection was lost due to an unknown reason
				InSim.Reconnected += new InSimInterface.Reconnected_EventHandler(ReconnectedToInSim);			// Occurs when connection was recovert automatically

				InitializeInSimEvents();				// Initialize packet receive events
				InSim.Connect();						// Attempt to connect to the InSim host 
			}
			catch (InvalidConfigurationException ex)
			{
				MessageBox.Show(ex.Message);
			}
			catch (UnableToConnectToInSimException ex)
			{
				MessageBox.Show(ex.Message);
			}
			catch (AdminPasswordDoesNotMatchException ex)
			{
				MessageBox.Show(ex.Message);
			}
			catch (UDPListenPortAlreadyInUseException ex)
			{
				MessageBox.Show(ex.Message);
			}
			catch (AutoReconnectFailedException ex)
			{
				MessageBox.Show(ex.Message);
			}
			catch (Exception ex)
			{
				MessageBox.Show(ex.Message);
			}
			finally
			{
				if (InSim.State == LFS_External.InSim.InSimInterface.InSimState.Connected)
				{
					// Request all players and connections so we can build the Connections and Players list
					InSim.Request_NCN_AllConnections(255);
					InSim.Request_NPL_AllPlayers(255);
				}
			}
		}

        private void PayUser_Elapsed(object sender, System.Timers.ElapsedEventArgs e)
        {
            foreach (clsPlayer Ply in Players)
            {
                if (Ply.Payout > 4)
                {
                    Connections[GetConnIdx(Ply.UniqueID)].Cash += 2;
                    Ply.Payout = 0;
                    InSim.Send_BTN_CreateButton("^7£" + Connections[GetConnIdx(Ply.UniqueID)].Cash, Flags.ButtonStyles.ISB_DARK, 5, 15, 0, 90, 5, Ply.UniqueID, 2, false);
                }
            }
        }

		// Occurs when connection was lost due to an unknown reason
		private void LostConnectionToInSim()
		{
			MessageBox.Show("Connection to InSim lost due to an unknown reason.");
		}

		// Occurs when connection was recovert automatically
		private void ReconnectedToInSim()
		{
			MessageBox.Show("Automatically reconnected to InSim.");
			// You should request all connections and players again so you can check changes in the corresponding lists
		}
		
		// You should only enable the events you need to gain maximum performance. All events are enable by default.
		private void InitializeInSimEvents()
		{
			// Client information
			InSim.NCN_Received += new LFS_External.InSim.InSimInterface.NCN_EventHandler(NCN_ClientJoinsHost);				// A new client joined the server.
			InSim.CNL_Received += new LFS_External.InSim.InSimInterface.CNL_EventHandler(CNL_ClientLeavesHost);				// A client left the server.
			InSim.CPR_Received += new LFS_External.InSim.InSimInterface.CPR_EventHandler(CPR_ClientRenames);				// A client changed name or plate.
			InSim.PFL_Received += new LFS_External.InSim.InSimInterface.PFL_EventHandler(PFL_PlayerFlagsChanged);			// Player help settings changed.
			InSim.PLP_Received += new LFS_External.InSim.InSimInterface.PLP_EventHandler(PLP_PlayerGoesToGarage);			// A player goes to the garage (setup screen).
			InSim.NPL_Received += new LFS_External.InSim.InSimInterface.NPL_EventHandler(NPL_PlayerJoinsRace);				// A player join the race. If PLID already exists, then player leaves pit.
			InSim.LAP_Received += new LFS_External.InSim.InSimInterface.LAP_EventHandler(LAP_PlayerCompletesLap);			// A player crosses start/finish line
			InSim.TOC_Received += new LFS_External.InSim.InSimInterface.TOC_EventHandler(TOC_PlayerCarTakeOver);			// Car got taken over by an other player
			InSim.FIN_Received += new LFS_External.InSim.InSimInterface.FIN_EventHandler(FIN_PlayerFinishedRaces);			// A player finishes race and crosses the finish line
			InSim.CRS_Received += new LFS_External.InSim.InSimInterface.CRS_EventHandler(CRS_PlayerResetsCar);				// A player resets the car
			InSim.PIT_Received += new LFS_External.InSim.InSimInterface.PIT_EventHandler(PIT_PlayerStopsAtPit);				// A player stops for making a pitstop
			InSim.PSF_Received += new LFS_External.InSim.InSimInterface.PSF_EventHandler(PSF_PitStopFinished);				// A pitstop got finished
			InSim.AXO_Received += new LFS_External.InSim.InSimInterface.AXO_EventHandler(AXO_PlayerHitsAutocrossObject);	// A player hits an autocross object
			InSim.PLL_Received += new LFS_External.InSim.InSimInterface.PLL_EventHandler(PLL_PlayerLeavesRace);				// A player leaves the race (spectate)
			InSim.BFN_Received += new LFS_External.InSim.InSimInterface.BFN_EventHandler(BFN_PlayerRequestsButtons);		// A player pressed Shift+I or Shift+B
			InSim.BTC_Received += new LFS_External.InSim.InSimInterface.BTC_EventHandler(BTC_ButtonClicked);				// A player clicked a custom button
			InSim.BTT_Received += new LFS_External.InSim.InSimInterface.BTT_EventHandler(BTT_TextBoxOkClicked);				// A player submitted a custom textbox
			InSim.PEN_Received += new LFS_External.InSim.InSimInterface.PEN_EventHandler(PEN_PenaltyChanged);				// A penalty give or cleared
			InSim.FLG_Received += new LFS_External.InSim.InSimInterface.FLG_EventHandler(FLG_FlagChanged);					// Yellow or blue flag changed
			InSim.PLA_Received += new LFS_External.InSim.InSimInterface.PLA_EventHandler(PLA_PitLaneChanged);				// A player entered or left the pitlane
			InSim.SPX_Received += new LFS_External.InSim.InSimInterface.SPX_EventHandler(SPX_SplitTime);					// A player crossed a lap split
			InSim.CCH_Received += new LFS_External.InSim.InSimInterface.CCH_EventHandler(CCH_CameraChanged);				// A player changed it's camera

			// Host and race information
			InSim.STA_Received += new LFS_External.InSim.InSimInterface.STA_EventHandler(STA_StateChanged);					// The server/race state changed
			InSim.ISM_Received += new LFS_External.InSim.InSimInterface.ISM_EventHandler(ISM_MultiplayerInformation);		// A host is started or joined
			InSim.MPE_Received += new LFS_External.InSim.InSimInterface.MPE_EventHandler(MPE_MultiplayerEnd);				// A host ends or leaves
			InSim.CLR_Received += new LFS_External.InSim.InSimInterface.CLR_EventHandler(CLR_RaceCleared);					// Race got cleared with /clear
			InSim.REO_Received += new LFS_External.InSim.InSimInterface.REO_EventHandler(REO_RaceStartOrder);				// Sent at the start of every race or qualifying session, listing the start order
			InSim.RST_Received += new LFS_External.InSim.InSimInterface.RST_EventHandler(RST_RaceStart);					// A race starts
			InSim.RES_Received += new LFS_External.InSim.InSimInterface.RES_EventHandler(RES_RaceOrQualifyingResult);		// Qualify or confirmed finish
			InSim.REN_Received += new LFS_External.InSim.InSimInterface.REN_EventHandler(REN_RaceEnds);						// A race ends (return to game setup screen)
			InSim.RTP_Received += new LFS_External.InSim.InSimInterface.RTP_EventHandler(RTP_RaceTime);						// Current race time progress in hundredths
			InSim.AXC_Received += new LFS_External.InSim.InSimInterface.AXC_EventHandler(AXC_AutocrossCleared);				// Autocross got cleared
			InSim.AXI_Received += new LFS_External.InSim.InSimInterface.AXI_EventHandler(AXI_AutocrossLayoutInformation);	// Request - autocross layout information
			InSim.CPP_Received += new LFS_External.InSim.InSimInterface.CPP_EventHandler(CPP_CameraPosition);				// LFS reporting camera position and state
			InSim.VTA_Received += new LFS_External.InSim.InSimInterface.VTA_EventHandler(VTA_VoteAction);					// A vote completed
			InSim.VTC_Received += new LFS_External.InSim.InSimInterface.VTC_EventHandler(VTC_VoteCanceled);					// A vote got canceled
			InSim.VTN_Received += new LFS_External.InSim.InSimInterface.VTN_EventHandler(VTN_VoteNotify);					// A vote got called

			// Car tracking
			InSim.MCI_Received += new LFS_External.InSim.InSimInterface.MCI_EventHandler(MCI_CarInformation);				// Detailed car information packet (max 8 per packet)
			InSim.NLP_Received += new LFS_External.InSim.InSimInterface.NLP_EventHandler(NLP_LapNode);						// Compact car information packet

			// Other
			InSim.MSO_Received += new LFS_External.InSim.InSimInterface.MSO_EventHandler(MSO_MessageOut);					// Player chat and system messages.
			InSim.III_Received += new LFS_External.InSim.InSimInterface.III_EventHandler(III_InSimInfo);					// A /i message got sent to this program
			InSim.VER_Received += new LFS_External.InSim.InSimInterface.VER_EventHandler(VER_InSimVersionInformation);		// InSim version information
			InSim.REPLY_Received += new LFS_External.InSim.InSimInterface.REPLY_EventHandler(REPLY_PingReplay);				// Reply to a ping request
		}

		#region ' Utils '
		// Methods for automatically update Players[] and Connection[] lists
		private void RemoveFromConnectionsList(byte ucid)
		{
			// Copy of item to remove
			clsConnection RemoveItem = new clsConnection();

			// Check what item the connection had
			foreach (clsConnection Conn in Connections)
			{
				if (ucid == Conn.UniqueID)
				{
					// Copy item (Can't delete it here)
					RemoveItem = Conn;
					continue;
				}
			}

			// Remove item
			Connections.Remove(RemoveItem);
		}
		private void AddToConnectionsList(Packets.IS_NCN NCN)
		{
			bool InList = false;

			// Check of connection is already in the list
			foreach (clsConnection Conn in Connections)
			{
				if (Conn.UniqueID == NCN.UCID)
				{
					InList = true;
					continue;
				}
			}

			// If not, add it
			if (!InList)
			{
				// Assign values of new connnnection.
				clsConnection NewConn = new clsConnection();
				NewConn.UniqueID = NCN.UCID;
				NewConn.Username = NCN.UName;
				NewConn.PlayerName = NCN.PName;
				NewConn.IsAdmin = NCN.Admin;
				NewConn.Flags = NCN.Flags;
                NewConn.Cars = FileInfo.GetUserCars(NCN.UName);
                NewConn.Cash = FileInfo.GetUserCash(NCN.UName);

				Connections.Add(NewConn);
			}
		}
		private void RemoveFromPlayersList(byte plid)
		{
			// Copy of item to remove
			clsPlayer RemoveItem = new clsPlayer();

			// Check what item the player had
			foreach (clsPlayer Player in Players)
			{
				if (plid == Player.PlayerID)
				{
					// Copy item (Can't delete it here)
					RemoveItem = Player;
					continue;
				}
			}

			// Remove item
			Players.Remove(RemoveItem);
		}
		private bool AddToPlayersList(Packets.IS_NPL NPL)
		{
			bool InList = false;

			// Check if player is already in the list
			foreach (clsPlayer Player in Players)
			{
				if (Player.PlayerID == NPL.PLID)
				{
					Player.AddedMass = NPL.H_Mass;
					Player.CarName = NPL.CName;
					Player.Flags = NPL.Flags;
					Player.Passengers = NPL.Pass;
					Player.Plate = NPL.Plate;
					Player.PlayerType = (clsPlayer.enuPType)NPL.PType;
					Player.SkinName = NPL.SName;
					Player.Tyre_FL = NPL.Tyre_FL;
					Player.Tyre_FR = NPL.Tyre_FR;
					Player.Tyre_RL = NPL.Tyre_RL;
					Player.Tyre_RR = NPL.Tyre_RR;
					Player.IntakeRestriction = NPL.H_TRes;
					return true;
				}
			}

			// If not, add it
			if (!InList)
			{
				// Assign values of new player.
				clsPlayer NewPlayer = new clsPlayer();
				NewPlayer.AddedMass = NPL.H_Mass;
				NewPlayer.CarName = NPL.CName;
				NewPlayer.Flags = NPL.Flags;
				NewPlayer.Passengers = NPL.Pass;
				NewPlayer.Plate = NPL.Plate;
				NewPlayer.PlayerID = NPL.PLID;
				NewPlayer.UniqueID = NPL.UCID;
				NewPlayer.PlayerName = NPL.PName;
				NewPlayer.PlayerType = (clsPlayer.enuPType)NPL.PType;
				NewPlayer.SkinName = NPL.SName;
				NewPlayer.Tyre_FL = NPL.Tyre_FL;
				NewPlayer.Tyre_FR = NPL.Tyre_FR;
				NewPlayer.Tyre_RL = NPL.Tyre_RL;
				NewPlayer.Tyre_RR = NPL.Tyre_RR;

				Players.Add(NewPlayer);
			}

			return false;
		}

		/// <summary>
		/// Returns an index value for Connections[] that corresponds with the UniqueID of a connection
		/// </summary>
		/// <param name="UNID">UCID to find</param>
		public int GetConnIdx(int UNID)
		{
			for (int i = 0; i < Connections.Count; i++)
			{
				if (Connections[i].UniqueID == UNID) { return i; }
			}
			return 0;
		}

		/// <summary>
		/// Returns an index value for Players[] that corresponds with the UniqueID of a player
		/// </summary>
		/// <param name="PLID">PLID to find</param>
		public int GetPlyIdx(int PLID)
		{
			for (int i = 0; i < Players.Count; i++)
			{
				if (Players[i].PlayerID == PLID) { return i; }
			}
			return 0;
		}

		/// <summary>Returns true if method needs invoking due to threading</summary>
		private bool DoInvoke()
		{
			foreach (Control c in this.Controls)
			{
				if (c.InvokeRequired) return true;
				break;	// 1 control is enough
			}
			return false;
		}

		#endregion

		#region ' Packet receive events '

		// Player chat and system messages.
		private void MSO_MessageOut(Packets.IS_MSO MSO)
		{
            string Msg = MSO.Msg.Substring(MSO.TextStart, (MSO.Msg.Length - MSO.TextStart));
            string[] StrMsg = Msg.Split(' ');

            switch (StrMsg[0])
            {
                case "!help":
                    InSim.Send_MTC_MessageToConnection("^7Help menu!", MSO.UCID, 0);
                    break;

                case "!cash":
                    InSim.Send_MTC_MessageToConnection("^7Cash: " + Connections[GetConnIdx(MSO.UCID)].Cash, MSO.UCID, 0);
                    break;

                case "!cars":
                    InSim.Send_MTC_MessageToConnection("^7Cars: " + Connections[GetConnIdx(MSO.UCID)].Cars, MSO.UCID, 0);
                    break;

                case "!pay":
                    if (StrMsg.Length > 0)
                    {
                        if (StrMsg[1].Contains("-"))
                            InSim.Send_MTC_MessageToConnection("^7Invalid command", MSO.UCID, 0);
                        else
                        {
                            int Pay = int.Parse(StrMsg[1]);
                            Connections[GetConnIdx(MSO.UCID)].Cash -= Pay;
                            InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + "^7 paid £" + Pay + " fine");
                        }
                    }
                    break;

                case "!buy":
                    if (StrMsg.Length > 1)
                    {
                        if (Connections[GetConnIdx(MSO.UCID)].Cars.Contains(StrMsg[1].ToUpper()))
                            InSim.Send_MTC_MessageToConnection("^7Car already owned", MSO.UCID, 0);
                        else if (Dealer.GetCarPrice(StrMsg[1]) == 0)
                            InSim.Send_MTC_MessageToConnection("^7Invalid car selection", MSO.UCID, 0);
                        else if (Connections[GetConnIdx(MSO.UCID)].Cash >= Dealer.GetCarPrice(StrMsg[1]))
                        {
                            string Cars = Connections[GetConnIdx(MSO.UCID)].Cars;

                            switch (StrMsg[1].ToUpper())
                            {
                                case "UF1":
                                    Cars = Cars + " " + "UF1";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("UF1");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a UF1");
                                    break;

                                case "XFG":
                                    Cars = Cars + " " + "XFG";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("XFG");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a XFG");
                                    break;

                                case "XRG":
                                    Cars = Cars + " " + "XRG";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("XRG");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a XRG");
                                    break;

                                case "LX4":
                                    Cars = Cars + " " + "LX4";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("LX4");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a LX4");
                                    break;

                                case "LX6":
                                    Cars = Cars + " " + "LX6";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("LX6");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a LX6");
                                    break;

                                case "RB4":
                                    Cars = Cars + " " + "RB4";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("RB4");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a RB4");
                                    break;

                                case "FXO":
                                    Cars = Cars + " " + "FXO";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("FXO");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a FXO");
                                    break;

                                case "XRT":
                                    Cars = Cars + " " + "XRT";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("XRT");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a XRT");
                                    break;

                                case "RAC":
                                    Cars = Cars + " " + "RAC";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("RAC");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a RAC");
                                    break;

                                case "FZ5":
                                    Cars = Cars + " " + "FZ5";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("FZ5");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a FZ5");
                                    break;

                                case "UFR":
                                    Cars = Cars + " " + "UFR";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("UFR");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a UFR");
                                    break;

                                case "XFR":
                                    Cars = Cars + " " + "XFR";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("XFR");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a XFR");
                                    break;

                                case "FXR":
                                    Cars = Cars + " " + "FXR";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("FXR");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a FXR");
                                    break;

                                case "XRR":
                                    Cars = Cars + " " + "XRR";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("XRR");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a XRR");
                                    break;

                                case "FZR":
                                    Cars = Cars + " " + "FZR";
                                    Connections[GetConnIdx(MSO.UCID)].Cars = Cars;
                                    Connections[GetConnIdx(MSO.UCID)].Cash -= Dealer.GetCarPrice("FZR");
                                    InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " ^7bought a FZR");
                                    break;
                            }
                        }
                        else InSim.Send_MTC_MessageToConnection("^7Not enough cash", MSO.UCID, 0);
                    }
                    else InSim.Send_MTC_MessageToConnection("^7Command error", MSO.UCID, 0);
                    break;


                case "!sell":
                    if (StrMsg.Length > 1)
                    {
                        if (Connections[GetConnIdx(MSO.UCID)].Cars.Contains(StrMsg[1].ToUpper()))
                        {
                            if (Dealer.GetCarPrice(StrMsg[1].ToUpper()) > 0)
                            {
                                string UserCars = Connections[GetConnIdx(MSO.UCID)].Cars.Trim();
                                int IdxCar = UserCars.IndexOf(StrMsg[1].ToUpper().Trim());
                                //string UserCars = Connections[GetConnIdx(MSO.UCID)].Cars;
                                try { Connections[GetConnIdx(MSO.UCID)].Cars = UserCars.Remove(IdxCar, 4); }
                                catch { Connections[GetConnIdx(MSO.UCID)].Cars = UserCars.Remove(IdxCar, 3); }
                                Connections[GetConnIdx(MSO.UCID)].Cash += Dealer.GetCarValue(StrMsg[1].ToUpper());
                                InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(MSO.UCID)].PlayerName + " sold " + StrMsg[1].ToUpper());

                            }
                            else InSim.Send_MTC_MessageToConnection("^7Car does not exist!", MSO.UCID, 0);
                        }
                        else InSim.Send_MTC_MessageToConnection("^7You do not own that car!", MSO.UCID, 0);
                    }
                    else InSim.Send_MTC_MessageToConnection("^7Command error!", MSO.UCID, 0);
                    break;

            }
		}

		// A new client joined the server.
		private void NCN_ClientJoinsHost(Packets.IS_NCN NCN)
		{
			AddToConnectionsList(NCN);	// Update Connections[] list
            FileInfo.NewUser(NCN.UName);
            InSim.Send_BTN_CreateButton("^7£" + Connections[GetConnIdx(NCN.UCID)].Cash, Flags.ButtonStyles.ISB_DARK, 5, 15, 0, 90, 5, NCN.UCID, 2, false);
			// Your code here
		}

		// A client left the server.
		private void CNL_ClientLeavesHost(Packets.IS_CNL CNL)
		{
            FileInfo.UpdateUserLeave(Connections[GetConnIdx(CNL.UCID)].Username, Connections[GetConnIdx(CNL.UCID)].Cash, Connections[GetConnIdx(CNL.UCID)].Cars);
			RemoveFromConnectionsList(CNL.UCID);		// Update Connections[] list
		}

		// A client changed name or plate.
		private void CPR_ClientRenames(Packets.IS_CPR CPR)
		{
			foreach (clsConnection c in Connections)	// Update Connections[] list
				if (c.UniqueID == CPR.UCID)
				{
					c.PlayerName = CPR.PName;
				}
			foreach (clsPlayer p in Players)	// Update Players[] list
				if (p.UniqueID == CPR.UCID)
				{
					p.PlayerName = CPR.PName;
					p.Plate = CPR.Plate;
				}
		}

		// Car was taken over by an other player
		private void TOC_PlayerCarTakeOver(Packets.IS_TOC TOC)
		{
			Players[GetPlyIdx(TOC.OldUCID)].UniqueID = TOC.NewUCID;	// Update Players[] list
			Players[GetPlyIdx(TOC.OldUCID)].PlayerID = TOC.PLID;	// Update Players[] list

			// Your code here
		}

		// A player leaves the race (spectate)
		private void PLL_PlayerLeavesRace(Packets.IS_PLL PLL)
		{
			RemoveFromPlayersList(PLL.PLID);		// Update Players[] list

			// Your code here
		}

		// Player help settings changed.
		private void PFL_PlayerFlagsChanged(Packets.IS_PFL PFL)
		{
		}

		// A player goes to the garage (setup screen).
		private void PLP_PlayerGoesToGarage(Packets.IS_PLP PLP)
		{
		}

		// A player joins the race. If PLID already exists, then player leaves pit.
		private void NPL_PlayerJoinsRace(Packets.IS_NPL NPL)
		{
			bool LeavesPits = AddToPlayersList(NPL);	// Update Players[] list
            if (Connections[GetConnIdx(NPL.UCID)].Cars.Contains(NPL.CName) != true)
            {
                InSim.Send_MST_Message("/spec " + Connections[GetConnIdx(NPL.UCID)].Username);
                InSim.Send_MST_Message("/msg " + Connections[GetConnIdx(NPL.UCID)].PlayerName + " ^7tried to steal " + NPL.CName);
            }
			// Your code here
		}

		// A player crosses start/finish line
		private void LAP_PlayerCompletesLap(Packets.IS_LAP LAP)
		{
		}

		// A player finishes race and crosses the finish line
		private void FIN_PlayerFinishedRaces(Packets.IS_FIN FIN)
		{
		}

		// A player resets the car
		private void CRS_PlayerResetsCar(Packets.IS_CRS CRS)
		{
		}

		// A player stops for making a pitstop
		private void PIT_PlayerStopsAtPit(Packets.IS_PIT PIT)
		{
		}

		// A pitstop got finished
		private void PSF_PitStopFinished(Packets.IS_PSF PSF)
		{
		}

		// A player hits an autocross object
		private void AXO_PlayerHitsAutocrossObject(Packets.IS_AXO AXO)
		{
		}

		// A player pressed Shift+I or Shift+B
		private void BFN_PlayerRequestsButtons(Packets.IS_BFN BFN)
		{
		}

		// A player clicked a custom button
		private void BTC_ButtonClicked(Packets.IS_BTC BTC)
		{
		}

		// A player submitted a custom textbox
		private void BTT_TextBoxOkClicked(Packets.IS_BTT BTT)
		{
		}

		// A penalty give or cleared
		private void PEN_PenaltyChanged(Packets.IS_PEN PEN)
		{
		}

		// Yellow or blue flag changed
		private void FLG_FlagChanged(Packets.IS_FLG FLG)
		{
		}

		// A player entered or left the pitlane
		private void PLA_PitLaneChanged(Packets.IS_PLA PLA)
		{
		}

		// A player crossed a lap split
		private void SPX_SplitTime(Packets.IS_SPX SPX)
		{
		}

		// A player changed it's camera
		private void CCH_CameraChanged(Packets.IS_CCH CCH)
		{
		}

		// The server/race state changed
		private void STA_StateChanged(Packets.IS_STA STA)
		{
		}

		// A host is started or joined
		private void ISM_MultiplayerInformation(Packets.IS_ISM ISM)
		{
		}

		// A host ends or leaves
		private void MPE_MultiplayerEnd()
		{
		}

		// Race got cleared with /clear
		private void CLR_RaceCleared()
		{
		}

		// Sent at the start of every race or qualifying session, listing the start order
		private void REO_RaceStartOrder(Packets.IS_REO REO)
		{
		}

		// Race start information
		private void RST_RaceStart(Packets.IS_RST RST)
		{
		}

		// Qualify or confirmed finish
		private void RES_RaceOrQualifyingResult(Packets.IS_RES RES)
		{
		}

		// A race ends (return to game setup screen)
		private void REN_RaceEnds()
		{
		}

		// Current race time progress in hundredths
		private void RTP_RaceTime(uint RTP)
		{
		}

		// Autocross got cleared
		private void AXC_AutocrossCleared()
		{
		}

		// Request - autocross layout information
		private void AXI_AutocrossLayoutInformation(Packets.IS_AXI AXI)
		{
		}

		// LFS reporting camera position and state
		private void CPP_CameraPosition(Packets.IS_CPP CPP)
		{
		}

		// A vote completed
		private void VTA_VoteAction(byte VTA)
		{
		}

		// A vote got canceled
		private void VTC_VoteCanceled()
		{
		}

		// A vote got called
		private void VTN_VoteNotify(Packets.IS_VTN VTN)
		{
		}

		// Detailed car information packet (max 8 per packet)
		private void MCI_CarInformation(Packets.IS_MCI MCI)
		{
            for (int i = 0; i < MCI.NumC; i++)
            {
                decimal Speed = (decimal)((MCI.Info[i].Speed * (100f / 32768f)) * 3.6f);
                decimal ConvSpeed = (decimal)(Speed * 25 / 1000);
                Players[GetPlyIdx(MCI.Info[i].PLID)].Payout += ConvSpeed;
            }
		}

		// Compact car information packet
		private void NLP_LapNode(Packets.IS_NLP NLP)
		{
		}

		// A /i message got sent to this program
		private void III_InSimInfo(Packets.IS_III III)
		{
		}

		// InSim version information
		private void VER_InSimVersionInformation(Packets.IS_VER VER)
		{
		}

		// Reply to a ping request
		private void REPLY_PingReplay()
		{
		}

		#endregion
	}
}
